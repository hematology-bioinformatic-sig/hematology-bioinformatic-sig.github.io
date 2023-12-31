---
title: 翻译|LC-MS/MS data analysis with xcms
author: zhengyanhua
date: '2023-08-11'
slug: []
categories: []
tags:
  - bioconductor workflow
Description: ''
Tags: []
Categories: []
DisableComments: no
---



# LC-MS/MS data analysis with xcms

## 1Introduction 介绍

Metabolite identification is an important step in non-targeted metabolomics and requires different steps. One involves the use of tandem mass spectrometry to generate fragmentation spectra of detected metabolites (LC-MS/MS), which are then compared to fragmentation spectra of known metabolites. Different approaches exist for the generation of these fragmentation spectra, whereas the most used is data dependent acquisition (DDA) also known as the top-n method. In this method the top N most intense m/z values from a MS1 scan are selected for fragmentation in the next N scans before the cycle starts again. This method allows to generate clean MS2 fragmentation spectra on the fly during acquisition without the need for further experiments, but suffers from poor coverage of the detected metabolites (since only a limited number of ions are fragmented).

> 代谢产物鉴定是非靶向代谢组学中的一个重要步骤，需要不同的步骤。其中之一涉及使用串联质谱法生成检测到的代谢物的裂解光谱（LC-MS/MS），然后将其与已知代谢物的裂解光谱进行比较。生成这些碎片谱的方法不同，而最常用的是数据相关采集（DDA），也称为top-n方法。在该方法中，在循环再次开始之前，从MS1扫描中选择前N个最密集的m/z值，以在接下来的N次扫描中进行碎片化。该方法允许在采集过程中动态生成干净的MS2碎片光谱，无需进一步实验，但检测到的代谢物覆盖率低（因为只有有限数量的离子被碎片化）。

Data independent approaches (DIA) like Bruker bbCID, Agilent AllIons or Waters MSe don’t use such a preselection, but rather fragment all detected molecules at once. They are using alternating schemes with scan of low and high collision energy to collect MS1 and MS2 data. Using this approach, there is no problem in coverage, but the relation between the precursor and fragment masses is lost leading to chimeric spectra. Sequential Window Acquisition of all Theoretical Mass Spectra (or SWATH [1]) combines both approaches through a middle-way approach. There is no precursor selection and acquisition is independent of acquired data, but rather than isolating all precusors at once, defined windows (i.e. ranges of m/z values) are used and scanned. This reduces the overlap of fragment spectra while still keeping a high coverage.

> 与数据无关的方法（DIA），如Bruker bbCID、安捷伦AllIons或Waters MSe，不使用这种预选，而是一次将所有检测到的分子片段化。他们使用交替方案，扫描低碰撞能量和高碰撞能量，以收集MS1和MS2数据。使用这种方法，覆盖率没有问题，但前驱体和片段质量之间的关系丢失，导致嵌合光谱。所有理论质谱的连续窗口获取（或SWATH[1]）通过中间方法将这两种方法结合起来。没有前兆选择，采集独立于采集的数据，而是使用和扫描定义的窗口（即m/z值的范围），而不是一次隔离所有前兆。这减少了片段光谱的重叠，同时仍保持高覆盖率。

This document showcases the analysis of two small LC-MS/MS data sets using xcms. The data files used are reversed-phase LC-MS/MS runs from the Agilent Pesticide mix obtained from a Sciex 6600 Triple ToF operated in SWATH acquisition mode. For comparison a DDA file from the same sample is included.

> 本文展示了使用xcms对两个小型LC-MS/MS数据集的分析。使用的数据文件是从安捷伦农药混合物中运行的反相LC-MS/MS，安捷伦农药混合物是从Sciex 6600三重ToF中获得的，该ToF在SWATH采集模式下运行。为了进行比较，包括来自同一样本的DDA文件。

## 2Analysis of DDA data 分析DDA数据

Below we load the example DDA data set using the readMSData function from the MSnbase package.

library(xcms)

dda_file <- system.file("TripleTOF-SWATH", "PestMix1_DDA.mzML",
                        package = "msdata")
dda_data <- readMSData(dda_file, mode = "onDisk")
The variable dda_data contains now all MS1 and MS2 spectra from the specified mzML file. The number of spectra for each MS level is listed below.

table(msLevel(dda_data))
## 
##    1    2 
## 1504 2238

For the MS2 spectra we can get the m/z of the precursor ion with the precursorMz function. Below we first filter the data set by MS level, extract the precursor m/z and call head to just show the first 6 elements. For easier readability we use the forward pipe operator %>% from the magrittr package.

> 对于MS2光谱，我们可以使用precursorMz函数获得前体离子的m/z。下面我们首先按MS级别过滤数据集，提取前驱体m/z并调用头以仅显示前6个元素。为了便于阅读，我们使用magrittr包中的正向管道操作符%>%。

library(magrittr)

dda_data %>%
    filterMsLevel(2L) %>%
    precursorMz() %>%
    head()
##  F1.S1570  F1.S1588  F1.S1592  F1.S1594  F1.S1595  F1.S1596 
## 130.96578 388.25426  89.93779  83.99569 371.22409 388.25226

With the precursorIntensity function it is also possible to extract the intensity of the precursor ion.

> 利用前兆强度函数也可以提取前体离子的强度。

dda_data %>%
    filterMsLevel(2L) %>%
    precursorIntensity() %>%
    head()
## F1.S1570 F1.S1588 F1.S1592 F1.S1594 F1.S1595 F1.S1596 
##        0        0        0        0        0        0

Some manufacturers (like Sciex for the present test data) don’t define/export the precursor intensity and thus either NA or 0 is reported. We can however use the estimatePrecursorIntensity function from the xcms package to determine the precursor intensity for a MS 2 spectrum based on the intensity of the respective ion in the previous MS1 scan (note that with method = "interpolation" the precursor intensity would be defined based on interpolation between the intensity in the previous and subsequent MS1 scan). Below we estimate the precursor intensities, on the full data (for MS1 spectra a NA value is reported). Note also that we use xcms:: to call the function from the xcms package, because a function with the same name is also implemented in the Spectra package, which would however not support OnDiskMSnExp objects as input.

> 一些制造商（如目前测试数据的Sciex）没有定义/输出前体强度，因此报告为NA或0。然而，我们可以使用xcms软件包中的estimatePrecursorIntensity函数，根据先前MS1扫描中各个离子的强度来确定MS 2光谱的前体强度（注意，使用方法=“插值”时，前体强度将根据先前和后续MS1扫描中的强度之间的插值来定义）。下面，我们根据完整数据估计前驱体强度（对于MS1光谱，报告了NA值）。还要注意，我们使用xcms：：从xcms包中调用函数，因为在Spectra包中也实现了一个同名函数，但它不支持将OnDiskMSnExp对象作为输入。

prec_int <- xcms::estimatePrecursorIntensity(dda_data)

We next set the precursor intensity in the spectrum metadata of dda_data. So that it can be extracted later with the precursorIntensity function.

> 接下来，我们在dda_数据的光谱元数据中设置前兆强度。这样以后就可以用前兆强度函数进行提取。

fData(dda_data)$precursorIntensity <- prec_int

dda_data %>%
    filterMsLevel(2L) %>%
    precursorIntensity() %>%
    head()
##  F1.S1570  F1.S1588  F1.S1592  F1.S1594  F1.S1595  F1.S1596 
## 0.9691072 3.0772917 0.3885723 0.3215049 1.6329483 4.4057989

Next we perform the chromatographic peak detection on the MS level 1 data with the findChromPeaks method. Below we define the settings for a centWave-based peak detection and perform the analysis.

> 接下来，我们使用findChromPeaks方法对MS 1级数据进行色谱峰检测。下面我们定义了基于centWave的峰值检测的设置，并进行了分析。

cwp <- CentWaveParam(snthresh = 5, noise = 100, ppm = 10,
                     peakwidth = c(3, 30))
dda_data <- findChromPeaks(dda_data, param = cwp)
In total 112 peaks were identified in the present data set.

The advantage of LC-MS/MS data is that (MS1) ions are fragmented and the corresponding MS2 spectra measured. Thus, for some of the ions (identified as MS1 chromatographic peaks) MS2 spectra are available. These can facilitate the annotation of the respective MS1 chromatographic peaks (or MS1 features after a correspondence analysis). Spectra for identified chromatographic peaks can be extracted with the chromPeakSpectra method. MS2 spectra with their precursor m/z and retention time within the rt and m/z range of the chromatographic peak are returned. Parameter return.type allows to define in which format these are returned. With return.type = "List" or return.type = "Spectra" the data is represented by a Spectra object from the Spectra.

> LC-MS/MS数据的优点是（MS1）离子被碎片化，并测量相应的MS2光谱。因此，对于某些离子（确定为MS1色谱峰），MS2光谱可用。这些可以帮助注释各自的MS1色谱峰（或对应分析后的MS1特征）。已识别色谱峰的光谱可以用色谱峰谱法提取。返回MS2光谱及其前体m/z和色谱峰的rt和m/z范围内的保留时间。参数返回。类型允许定义返回的格式。返回。键入=“List”或return。type=“光谱”数据由光谱中的光谱对象表示。

library(Spectra)
dda_spectra <- chromPeakSpectra(
    dda_data, msLevel = 2L, return.type = "Spectra")
dda_spectra
## MSn data (Spectra) with 150 spectra in a MsBackendMzR backend:
##            msLevel     rtime scanIndex
##          <integer> <numeric> <integer>
## F1.S1812         2   237.869      1812
## F1.S1846         2   241.299      1846
## F1.S2446         2   326.583      2446
## F1.S2450         2   327.113      2450
## F1.S2502         2   330.273      2502
## ...            ...       ...       ...
## F1.S5110         2   574.725      5110
## F1.S5115         2   575.255      5115
## F1.S5272         2   596.584      5272
## F1.S5236         2   592.424      5236
## F1.S5266         2   596.054      5266
##  ... 38 more variables/columns.
## 
## file(s):
## PestMix1_DDA.mzML

By default chromPeakSpectra returns all spectra associated with a MS1 chromatographic peak, but parameter method allows to choose and return only one spectrum per peak (have a look at the ?chromPeakSpectra help page for more details). Also, it would be possible to extract MS1 spectra for each peak by specifying msLevel = 1L in the call above (e.g. to evaluate the full MS1 signal at the peak’s apex position).

> 默认情况下，chromPeakSpectra返回与MS1色谱峰相关的所有光谱，但参数方法允许每个峰只选择和返回一个光谱（有关更多详细信息，请参阅？chromPeakSpectra帮助页面）。此外，可以通过在上述调用中指定msLevel=1L来提取每个峰值的MS1频谱（例如，评估峰值顶点位置的完整MS1信号）。

In the example above we selected to return the data as a Spectra object. Spectra variables "peak_id" and "peak_index" contain the identifiers and the index (in the chromPeaks matrix) of the chromatographic peaks the MS2 spectrum is associated with.

> 在上面的示例中，我们选择将数据作为光谱对象返回。光谱变量“peak\u id”和“peak\u index”包含MS2光谱相关色谱峰的标识符和索引（在色谱峰矩阵中）。

dda_spectra$peak_id
##   [1] "CP004" "CP004" "CP005" "CP006" "CP006" "CP008" "CP008" "CP011" "CP011"
##  [10] "CP012" "CP012" "CP013" "CP013" "CP013" "CP013" "CP014" "CP014" "CP014"
##  [19] "CP014" "CP018" "CP022" "CP022" "CP022" "CP022" "CP025" "CP025" "CP025"
##  [28] "CP025" "CP026" "CP026" "CP026" "CP026" "CP033" "CP033" "CP034" "CP034"
##  [37] "CP034" "CP034" "CP034" "CP035" "CP035" "CP035" "CP041" "CP041" "CP041"
##  [46] "CP042" "CP042" "CP042" "CP042" "CP042" "CP043" "CP048" "CP048" "CP050"
##  [55] "CP050" "CP050" "CP050" "CP051" "CP051" "CP051" "CP052" "CP052" "CP052"
##  [64] "CP054" "CP055" "CP055" "CP056" "CP056" "CP056" "CP057" "CP057" "CP057"
##  [73] "CP057" "CP057" "CP061" "CP061" "CP061" "CP061" "CP065" "CP065" "CP066"
##  [82] "CP066" "CP067" "CP067" "CP069" "CP069" "CP069" "CP069" "CP070" "CP070"
##  [91] "CP070" "CP070" "CP070" "CP072" "CP072" "CP072" "CP073" "CP074" "CP074"
## [100] "CP074" "CP074" "CP075" "CP075" "CP075" "CP077" "CP077" "CP077" "CP079"
## [109] "CP079" "CP079" "CP079" "CP080" "CP080" "CP080" "CP081" "CP087" "CP087"
## [118] "CP087" "CP087" "CP087" "CP089" "CP089" "CP089" "CP090" "CP090" "CP090"
## [127] "CP092" "CP092" "CP094" "CP094" "CP095" "CP095" "CP095" "CP096" "CP096"
## [136] "CP096" "CP097" "CP097" "CP097" "CP099" "CP099" "CP099" "CP099" "CP099"
## [145] "CP100" "CP100" "CP100" "CP101" "CP102" "CP102"

Note also that with return.type = "List" a list parallel to the chromPeaks matrix would be returned, i.e. each element in that list would contain the spectra for the chromatographic peak with the same index. This data representation might eventually simplify further processing.

> 还要注意的是，返回。type=“List”将返回与色谱峰矩阵平行的列表，即该列表中的每个元素将包含具有相同索引的色谱峰的光谱。这种数据表示可能最终简化进一步的处理。

We next use the MS2 information to aid in the annotation of a chromatographic peak. As an example we use a chromatographic peak of an ion with an m/z of 304.1131 which we extract in the code block below.

> 接下来，我们使用MS2信息来帮助注释色谱峰。例如，我们使用m/z为304.1131的离子色谱峰，我们在下面的代码块中提取该色谱峰。

ex_mz <- 304.1131
chromPeaks(dda_data, mz = ex_mz, ppm = 20)
##             mz    mzmin    mzmax      rt   rtmin   rtmax    into     intb
## CP057 304.1133 304.1126 304.1143 425.024 417.985 441.773 13040.8 12884.14
##           maxo sn sample
## CP057 3978.987 79      1

A search of potential ions with a similar m/z in a reference database (e.g. Metlin) returned a large list of potential hits, most with a very small ppm. For two of the hits, Flumazenil (Metlin ID 2724) and Fenamiphos (Metlin ID 72445) experimental MS2 spectra are available. Thus, we could match the MS2 spectrum for the identified chromatographic peak against these to annotate our ion. Below we extract all MS2 spectra that were associated with the candidate chromatographic peak using the ID of the peak in the present data set.

> 在参考数据库（例如Metlin）中搜索具有类似m/z的潜在离子时，返回了一个巨大的潜在点击列表，其中大多数具有非常小的ppm。对于其中两个点击，氟马西尼（Metlin ID 2724）和非那米磷（Metlin ID 72445）实验MS2光谱可用。因此，我们可以将已识别色谱峰的MS2光谱与这些色谱峰进行匹配，以注释我们的离子。下面，我们使用当前数据集中的峰ID提取与候选色谱峰相关的所有MS2光谱。

ex_id <- rownames(chromPeaks(dda_data, mz = ex_mz, ppm = 20))
ex_spectra <- dda_spectra[dda_spectra$peak_id == ex_id]
ex_spectra
## MSn data (Spectra) with 5 spectra in a MsBackendMzR backend:
##            msLevel     rtime scanIndex
##          <integer> <numeric> <integer>
## F1.S3505         2   418.926      3505
## F1.S3510         2   419.306      3510
## F1.S3582         2   423.036      3582
## F1.S3603         2   423.966      3603
## F1.S3609         2   424.296      3609
##  ... 38 more variables/columns.
## 
## file(s):
## PestMix1_DDA.mzML

There are 5 MS2 spectra representing fragmentation of the ion(s) measured in our candidate chromatographic peak. We next reduce this to a single MS2 spectrum using the combineSpectra method employing the combinePeaks function to determine which peaks to keep in the resulting spectrum (have a look at the ?combinePeaks help page for details). Parameter f allows to specify which spectra in the input object should be combined into one.

> 有5个MS2光谱代表在我们的候选色谱峰中测量的离子碎片。接下来，我们使用combinePeaks函数的combinePeaks方法将其减少为单个MS2频谱，以确定在结果频谱中保留哪些峰值（有关详细信息，请参阅？combinePeaks帮助页面）。参数f允许指定应将输入对象中的哪些光谱合并为一个。

ex_spectrum <- combineSpectra(ex_spectra, FUN = combinePeaks, ppm = 20,
                              peaks = "intersect", minProp = 0.8,
                              intensityFun = median, mzFun = median,
                              f = ex_spectra$peak_id)
ex_spectrum
## MSn data (Spectra) with 1 spectra in a MsBackendDataFrame backend:
##            msLevel     rtime scanIndex
##          <integer> <numeric> <integer>
## F1.S3505         2   418.926      3505
##  ... 38 more variables/columns.
## Processing:
##  Switch backend from MsBackendMzR to MsBackendDataFrame [Tue Apr 26 18:25:30 2022]

Mass peaks from all input spectra with a difference in m/z smaller 20 ppm (parameter ppm) were combined into one peak and the median m/z and intensity is reported for these. Due to parameter minProp = 0.8, the resulting MS2 spectrum contains only peaks that were present in 80% of the input spectra.

> 将m/z差小于20 ppm（参数ppm）的所有输入光谱的质量峰合并为一个峰，并报告这些峰的中值m/z和强度。由于参数minProp=0.8，得到的MS2光谱仅包含80%输入光谱中存在的峰值。

A plot of this consensus spectrum is shown below.

plotSpectra(ex_spectrum)

Consensus MS2 spectrum created from all measured MS2 spectra for ions of chromatographic peak CP53.

> 根据色谱峰CP53离子的所有测量MS2光谱创建共识MS2光谱。

We could now match the consensus spectrum against a database of MS2 spectra. In our example we simply load MS2 spectra for the two compounds with matching m/z exported from Metlin. For each of the compounds MS2 spectra created with collision energies of 0V, 10V, 20V and 40V are available. Below we import the respective data and plot our candidate spectrum against the MS2 spectra of Flumanezil and Fenamiphos (from a collision energy of 20V). To import files in MGF format we have to load the MsBackendMgf R package which adds MGF file support to the Spectra package.

> 我们现在可以将共识光谱与MS2光谱数据库进行匹配。在我们的示例中，我们只需加载两种化合物的MS2光谱，并从Metlin导出匹配的m/z。对于每种化合物，可以获得碰撞能量为0V、10V、20V和40V的MS2光谱。下面，我们导入各自的数据，并根据氟马奈齐和非那米磷的MS2光谱绘制候选光谱（来自20V碰撞能量）。要导入MGF格式的文件，我们必须加载MsBackendMgf R包，该包将MGF文件支持添加到Spectrum包中。

This package can be installed with BiocManager::install("RforMassSpectrometry/MsBackendMgf").

Prior plotting we normalize our experimental spectra.

norm_fun <- function(z, ...) {
    z[, "intensity"] <- z[, "intensity"] /
        max(z[, "intensity"], na.rm = TRUE) * 100
    z
}
ex_spectrum <- addProcessing(ex_spectrum, FUN = norm_fun)
library(MsBackendMgf)
flumanezil <- Spectra(
    system.file("mgf", "metlin-2724.mgf", package = "xcms"),
    source = MsBackendMgf())
## Start data import from 1 files ... done
fenamiphos <- Spectra(
    system.file("mgf", "metlin-72445.mgf", package = "xcms"),
    source = MsBackendMgf())
## Start data import from 1 files ... done
par(mfrow = c(1, 2))
plotSpectraMirror(ex_spectrum, flumanezil[3], main = "against Flumanezil",
                  ppm = 40)
plotSpectraMirror(ex_spectrum, fenamiphos[3], main = "against Fenamiphos",
                  ppm = 40)
                  
Mirror plots for the candidate MS2 spectrum against Flumanezil (left) and Fenamiphos (right). The upper panel represents the candidate MS2 spectrum, the lower the target MS2 spectrum. Matching peaks are indicated with a dot.

> 针对氟马奈齐（左）和非那米磷（右）的候选MS2光谱镜像图。上部面板表示候选MS2频谱，目标MS2频谱的下部。匹配峰值用点表示。

Our candidate spectrum matches Fenamiphos, thus, our example chromatographic peak represents signal measured for this compound. In addition to plotting the spectra, we can also calculate similarities between them with the compareSpectra method (which uses by default the normalized dot-product to calculate the similarity).

> 我们的候选光谱与非那米磷相匹配，因此，我们的示例色谱峰代表该化合物的测量信号。除了绘制光谱外，我们还可以使用compareSpectra方法（默认情况下使用归一化点积计算相似性）计算它们之间的相似性。

compareSpectra(ex_spectrum, flumanezil, ppm = 40)
## [1] 4.520957e-02 3.283806e-02 2.049379e-03 3.374354e-05
compareSpectra(ex_spectrum, fenamiphos, ppm = 40)
## [1] 0.1326234432 0.4879399946 0.7198406271 0.3997922658 0.0004876129
## [6] 0.0028408885 0.0071030051 0.0053809736

Clearly, the candidate spectrum does not match Flumanezil, while it has a high similarity to Fenamiphos. While we performed here the MS2-based annotation on a single chromatographic peak, this could be easily extended to the full list of MS2 spectra (returned by chromPeakSpectra) for all chromatographic peaks in an experiment. See also here.

> 显然，候选谱与氟马奈齐不匹配，但与非那米磷高度相似。虽然我们在这里对单个色谱峰进行了基于MS2的注释，但这可以很容易地扩展到实验中所有色谱峰的MS2光谱的完整列表（由chromPeakSpectra返回）。请参见此处。

In the present example we used only a single data file and we did thus not need to perform a sample alignment and correspondence analysis. These tasks could however be performed similarly to plain LC-MS data, retention times of recorded MS2 spectra would however also be adjusted during alignment based on the MS1 data. After correspondence analysis (peak grouping) MS2 spectra for features can be extracted with the featureSpectra function which returns all MS2 spectra associated with any chromatographic peak of a feature.

> 在本例中，我们仅使用单个数据文件，因此不需要执行样本对齐和对应分析。然而，这些任务可以类似于普通LC-MS数据来执行，然而，记录的MS2光谱的保留时间也将在基于MS1数据的校准期间进行调整。在对应分析（峰分组）后，可以使用FeatureSpectrum函数提取特征的MS2光谱，该函数返回与特征的任何色谱峰相关的所有MS2光谱。

Note also that this workflow can be included into the Feature-Based Molecular Networking FBMN to match MS2 spectra against GNPS. See here for more details and examples.

> 还请注意，该工作流可以包括在基于特征的分子网络FBMN中，以匹配MS2光谱与GNPS。有关更多详细信息和示例，请参见此处。

## 3SWATH data analysis

In this section we analyze a small SWATH data set consisting of a single mzML file with data from the same sample analyzed in the previous section but recorded in SWATH mode. We again read the data with the readMSData function. The resulting object will contain all recorded MS1 and MS2 spectra in the specified file.

> 在本节中，我们分析了一个小面积数据集，该数据集由单个mzML文件组成，数据来自上一节中分析的相同样本，但以小面积模式记录。我们再次使用readMSData函数读取数据。生成的对象将包含指定文件中记录的所有MS1和MS2光谱。

swath_file <- system.file("TripleTOF-SWATH",
                          "PestMix1_SWATH.mzML",
                          package = "msdata")

swath_data <- readMSData(swath_file, mode = "onDisk")
Below we determine the number of MS level 1 and 2 spectra in the present data set.

table(msLevel(swath_data))
## 
##    1    2 
##  444 3556

As described in the introduction, in SWATH mode all ions within pre-defined isolation windows are fragmented and MS2 spectra measured. The definition of these isolation windows (SWATH pockets) is imported from the mzML files and stored in the object’s fData (which provides additional annotations for each individual spectrum). Below we inspect the respective information for the first few spectra. The upper and lower isolation window m/z can be extracted with the isolationWindowLowerMz and isolationWindowUpperMz.

> 如引言所述，在SWATH模式下，预定义隔离窗口内的所有离子都被碎片化，并测量MS2光谱。这些隔离窗口（线束袋）的定义从mzML文件中导入，并存储在对象的fData中（该fData为每个单独的频谱提供额外的注释）。下面我们检查前几个光谱的相应信息。可以使用isolationWindowLowerMz和isolationWindowUpperMz提取上下隔离窗口m/z。

head(fData(swath_data)[, c("isolationWindowTargetMZ",
                           "isolationWindowLowerOffset",
                           "isolationWindowUpperOffset",
                           "msLevel", "retentionTime")])
##          isolationWindowTargetMZ isolationWindowLowerOffset
## F1.S2000                  208.95                      21.95
## F1.S2001                  244.05                      14.15
## F1.S2002                  270.85                      13.65
## F1.S2003                  299.10                      15.60
## F1.S2004                  329.80                      16.10
## F1.S2005                  367.35                      22.45
##          isolationWindowUpperOffset msLevel retentionTime
## F1.S2000                      21.95       2       200.084
## F1.S2001                      14.15       2       200.181
## F1.S2002                      13.65       2       200.278
## F1.S2003                      15.60       2       200.375
## F1.S2004                      16.10       2       200.472
## F1.S2005                      22.45       2       200.569
head(isolationWindowLowerMz(swath_data))
## [1] 187.0 229.9 257.2 283.5 313.7 344.9
head(isolationWindowUpperMz(swath_data))
## [1] 230.9 258.2 284.5 314.7 345.9 389.8

In the present data set we use the value of the isolation window target m/z to define the individual SWATH pockets. Below we list the number of spectra that are recorded in each pocket/isolation window.

> 在当前数据集中，我们使用隔离窗口目标m/z的值来定义单个线束袋。下面我们列出了每个口袋/隔离窗口中记录的光谱数量。

table(isolationWindowTargetMz(swath_data))
## 
## 163.75 208.95 244.05 270.85  299.1  329.8 367.35 601.85 
##    444    445    445    445    445    444    444    444
We have thus 1,000 MS2 spectra measured in each isolation window.

### 3.1Chromatographic peak detection in MS1 and MS2 data

Similar to a conventional LC-MS analysis, we perform first a chromatographic peak detection (on the MS level 1 data) with the findChromPeaks method. Below we define the settings for a centWave-based peak detection and perform the analysis.

> 与传统的LC-MS分析类似，我们首先使用findChromPeaks方法进行色谱峰检测（在MS 1级数据上）。下面我们定义了基于centWave的峰值检测的设置，并进行了分析。

cwp <- CentWaveParam(snthresh = 5, noise = 100, ppm = 10,
                     peakwidth = c(3, 30))
swath_data <- findChromPeaks(swath_data, param = cwp)

Next we perform a chromatographic peak detection in the MS level 2 data of each isolation window. We use the findChromPeaksIsolationWindow function employing the same peak detection algorithm reducing however the required signal-to-noise ratio. The isolationWindow parameter allows to specify which MS2 spectra belong to which isolation window and hence defines in which set of MS2 spectra chromatographic peak detection should be performed. While the default value for this parameter uses isolation windows provided by calling isolationWindowTargetMz on the object, it would also be possible to manually define the isolation windows, e.g. if the corresponding information is not available in the input mzML files.

> 接下来，我们在每个隔离窗口的MS 2级数据中执行色谱峰检测。我们使用FindChromePeaksIsolationWindow函数，采用相同的峰值检测算法，但降低了所需的信噪比。isolationWindow参数允许指定哪些MS2光谱属于哪个隔离窗口，从而定义应在哪组MS2光谱中执行色谱峰检测。虽然此参数的默认值使用通过在对象上调用isolationWindowTargetMz提供的隔离窗口，但也可以手动定义隔离窗口，例如，如果输入mzML文件中没有相应的信息。

cwp <- CentWaveParam(snthresh = 3, noise = 10, ppm = 10,
                     peakwidth = c(3, 30))
swath_data <- findChromPeaksIsolationWindow(swath_data, param = cwp)

The findChromPeaksIsolationWindow function added all peaks identified in the individual isolation windows to the chromPeaks matrix containing already the MS1 chromatographic peaks. These newly added peaks can be identified by the value of the "isolationWindow" column in the corresponding row in chromPeakData, which lists also the MS level in which the peak was identified.

> FindChromePeaksIsolationWindow函数将各个隔离窗口中识别的所有峰添加到已经包含MS1色谱峰的ChromePeaks矩阵中。这些新添加的峰可以通过chromPeakData中相应行中“isolationWindow”列的值来识别，该列还列出了识别峰的MS水平。

chromPeakData(swath_data)
## DataFrame with 368 rows and 6 columns
##        ms_level is_filled isolationWindow isolationWindowTargetMZ
##       <integer> <logical>        <factor>               <numeric>
## CP01          1     FALSE              NA                      NA
## CP02          1     FALSE              NA                      NA
## CP03          1     FALSE              NA                      NA
## CP04          1     FALSE              NA                      NA
## CP05          1     FALSE              NA                      NA
## ...         ...       ...             ...                     ...
## CP364         2     FALSE          601.85                  601.85
## CP365         2     FALSE          601.85                  601.85
## CP366         2     FALSE          601.85                  601.85
## CP367         2     FALSE          601.85                  601.85
## CP368         2     FALSE          601.85                  601.85
##       isolationWindowLowerMz isolationWindowUpperMz
##                    <numeric>              <numeric>
## CP01                      NA                     NA
## CP02                      NA                     NA
## CP03                      NA                     NA
## CP04                      NA                     NA
## CP05                      NA                     NA
## ...                      ...                    ...
## CP364                  388.8                  814.9
## CP365                  388.8                  814.9
## CP366                  388.8                  814.9
## CP367                  388.8                  814.9
## CP368                  388.8                  814.9

Below we count the number of chromatographic peaks identified within each isolation window (the number of chromatographic peaks identified in MS1 is 62).

> 下面我们统计每个隔离窗口内识别的色谱峰数（MS1中识别的色谱峰数为62）。

table(chromPeakData(swath_data)$isolationWindow)
## 
## 163.75 208.95 244.05 270.85  299.1  329.8 367.35 601.85 
##      2     38     32     14    105     23     61     31

We thus successfully identified chromatographic peaks in the different MS levels and isolation windows, but don’t have any actual MS2 spectra yet. These have to be reconstructed from the available chromatographic peak data which we will done in the next section.

> 因此，我们成功地在不同的质谱水平和分离窗口中识别出色谱峰，但还没有任何实际的MS2光谱。必须根据可用色谱峰数据重建这些数据，我们将在下一节中进行。

### 3.2Reconstruction of MS2 spectra

Identifying the signal of the fragment ions for the precursor measured by each MS1 chromatographic peak is a non-trivial task. The MS2 spectrum of the fragment ion for each MS1 chromatographic peak has to be reconstructed from the available MS2 signal (i.e. the chromatographic peaks identified in MS level 2). For SWATH data, fragment ion signal should be present in the isolation window that contains the m/z of the precursor ion and the chromatographic peak shape of the MS2 chromatographic peaks of fragment ions of a specific precursor should have a similar retention time and peak shape than the precursor’s MS1 chromatographic peak.

> 识别每个MS1色谱峰测量的前体碎片离子的信号是一项非常重要的任务。每个MS1色谱峰的碎片离子的MS2光谱必须根据可用的MS2信号重建（即MS level 2中确定的色谱峰）。对于SWATH数据，碎片离子信号应出现在包含前体离子m/z的隔离窗口中，并且特定前体碎片离子的MS2色谱峰的色谱峰形状应具有与前体的MS1色谱峰类似的保留时间和峰形状。


After detection of MS1 and MS2 chromatographic peaks has been performed, we can reconstruct the MS2 spectra using the reconstructChromPeakSpectra function. This function defines an MS2 spectrum for each MS1 chromatographic peak based on the following approach:

> 检测到MS1和MS2色谱峰后，我们可以使用重建色度峰谱函数重建MS2光谱。该函数基于以下方法为每个MS1色谱峰定义MS2光谱：

Identify MS2 chromatographic peaks in the isolation window containing the m/z of the ion (the MS1 chromatographic peak) that have approximately the same retention time than the MS1 chromatographic peak (the accepted difference in retention time can be defined with the diffRt parameter).
Extract the MS1 chromatographic peak and all MS2 chromatographic peaks identified by the previous step and correlate the peak shapes of the candidate MS2 chromatographic peaks with the shape of the MS1 peak. MS2 chromatographic peaks with a correlation coefficient larger than minCor are retained.
Reconstruct the MS2 spectrum using the m/z of all above selected MS2 chromatographic peaks and their intensity; each MS2 chromatographic peak selected for an MS1 peak will thus represent one mass peak in the reconstructed spectrum.

> 在包含离子m/z的隔离窗口中识别MS2色谱峰（MS1色谱峰），其保留时间约等于MS1色谱峰（保留时间的可接受差异可通过衍射参数定义）。
提取MS1色谱峰和上一步识别的所有MS2色谱峰，并将候选MS2色谱峰的峰形状与MS1峰的形状相关联。保留了相关系数大于minCor的MS2色谱峰。
使用上述所有选定MS2色谱峰的m/z及其强度重建MS2光谱；因此，为MS1峰选择的每个MS2色谱峰将代表重构光谱中的一个质量峰。

To illustrate this process we perform the individual steps on the example of Fenamiphos (exact mass 303.105800777 and m/z of [M+H]+ adduct 304.113077). As a first step we extract the chromatographic peak for this ion.

fenamiphos_mz <- 304.113077
fenamiphos_ms1_peak <- chromPeaks(swath_data, mz = fenamiphos_mz, ppm = 2)
fenamiphos_ms1_peak
##            mz    mzmin    mzmax      rt   rtmin   rtmax     into     intb
## CP34 304.1124 304.1121 304.1126 423.945 419.445 428.444 10697.34 10688.34
##          maxo  sn sample
## CP34 2401.849 618      1

Next we identify all MS2 chromatographic peaks that were identified in the isolation window containing the m/z of Fenamiphos. The information on the isolation window in which a chromatographic peak was identified is available in the chromPeakData (which contains arbitrary additional annotations to each individual chromatographic peak).

> 接下来，我们确定了在含有灭胺磷m/z的隔离窗口中确定的所有MS2色谱峰。色谱峰识别的隔离窗口信息可在ChromePeakData中找到（该数据包含每个单独色谱峰的任意附加注释）。

keep <- chromPeakData(swath_data)$isolationWindowLowerMz < fenamiphos_mz &
        chromPeakData(swath_data)$isolationWindowUpperMz > fenamiphos_mz
We also require the retention time of the MS2 chromatographic peaks to be similar to the retention time of the MS1 peak and extract the corresponding peak information.

keep <- keep &
    chromPeaks(swath_data)[, "rtmin"] < fenamiphos_ms1_peak[, "rt"] &
    chromPeaks(swath_data)[, "rtmax"] > fenamiphos_ms1_peak[, "rt"]

fenamiphos_ms2_peak <- chromPeaks(swath_data)[which(keep), ]

In total 24 MS2 chromatographic peaks match all the above condition. Next we extract their corresponding ion chromatograms, as well as the ion chromatogram of the MS1 peak. In addition we have to filter the object first by isolation window, keeping only spectra that were measured in that specific window and to specify to extract the chromatographic data from MS2 spectra (with msLevel = 2L).

> 总共24个MS2色谱峰符合上述所有条件。接下来，我们提取其相应的离子色谱图，以及MS1峰的离子色谱图。此外，我们必须首先通过隔离窗口过滤对象，仅保留在该特定窗口中测量的光谱，并指定从MS2光谱中提取色谱数据（msLevel=2L）。

rtr <- fenamiphos_ms1_peak[, c("rtmin", "rtmax")]
mzr <- fenamiphos_ms1_peak[, c("mzmin", "mzmax")]
fenamiphos_ms1_chr <- chromatogram(swath_data, rt = rtr, mz = mzr)

rtr <- fenamiphos_ms2_peak[, c("rtmin", "rtmax")]
mzr <- fenamiphos_ms2_peak[, c("mzmin", "mzmax")]
fenamiphos_ms2_chr <- chromatogram(
    filterIsolationWindow(swath_data, mz = fenamiphos_mz),
    rt = rtr, mz = mzr, msLevel = 2L)

We can now plot the extracted ion chromatogram of the MS1 and the extracted MS2 data.

plot(rtime(fenamiphos_ms1_chr[1, 1]),
     intensity(fenamiphos_ms1_chr[1, 1]),
     xlab = "retention time [s]", ylab = "intensity", pch = 16,
     ylim = c(0, 5000), col = "blue", type = "b", lwd = 2)
#' Add data from all MS2 peaks
tmp <- lapply(fenamiphos_ms2_chr@.Data,
              function(z) points(rtime(z), intensity(z),
                                 col = "#00000080",
                                 type = "b", pch = 16))
                                 
                                 
Extracted ion chromatograms for Fenamiphos from MS1 (blue) and potentially related signal in MS2 (grey).

> 从MS1（蓝色）和MS2（灰色）中潜在相关信号中提取非那米磷的离子色谱图。


Next we can calculate correlations between the peak shapes of each MS2 chromatogram with the MS1 peak. We perform the correlation below for one of the MS2 chromatographic peaks. Note that, because spectra are recorded consecutively, the retention times of the individual data points will differ for the MS2 and MS1 chromatographic data and data points have thus to be matched (aligned) before performing the correlation analysis. This is done automatically by the correlate function. See the help for the align method for more information on alignment options.

> 接下来，我们可以计算每个MS2色谱图的峰形状与MS1峰之间的相关性。我们对其中一个MS2色谱峰进行以下关联。注意，由于光谱是连续记录的，MS2和MS1色谱数据的各个数据点的保留时间将不同，因此在执行相关分析之前，必须匹配（对齐）数据点。这是由correlate函数自动完成的。有关对齐选项的详细信息，请参阅对齐方法的帮助。

correlate(fenamiphos_ms2_chr[1, 1],
          fenamiphos_ms1_chr[1, 1], align = "approx")
## Warning: '.local' is deprecated.
## Use 'compareChromatograms' instead.
## See help("Deprecated")
## [1] 0.9997871


After identifying the MS2 chromatographic peaks with shapes of enough high similarity to the MS1 chromatographic peaks, an MS2 spectrum could be reconstructed based on the m/z and intensities of the MS2 chromatographic peaks.

> 在识别出形状与MS1色谱峰高度相似的MS2色谱峰后，可以根据MS2色谱峰的m/z和强度重建MS2光谱。

The reconstructChromPeakSpectra function performs the above analysis for each individual MS1 chromatographic peak in a SWATH data set. Below we reconstruct MS2 spectra for our example data requiring a peak shape correlation higher than 0.9 between the candidate MS2 chromatographic peak and the target MS1 chromatographic peak. Again, we use return.type = "Spectra" to return the results as a Spectra object (instead to the default, but older/obsolete MSpectra object).

> 重建色度峰值谱函数对SWATH数据集中的每个单独MS1色谱峰执行上述分析。下面，我们为我们的示例数据重建MS2光谱，该示例数据要求候选MS2色谱峰和目标MS1色谱峰之间的峰形相关性高于0.9。同样，我们使用return。键入=“Spectra”以将结果作为Spectra对象返回（改为默认值，但为较旧/过时的MSSpectra对象）。

swath_spectra <- reconstructChromPeakSpectra(swath_data, minCor = 0.9,
                                             return.type = "Spectra")
swath_spectra
## MSn data (Spectra) with 62 spectra in a MsBackendDataFrame backend:
##       msLevel     rtime scanIndex
##     <integer> <numeric> <integer>
## 1           2   239.458        NA
## 2           2   240.358        NA
## 3           2   329.577        NA
## 4           2   329.771        NA
## 5           2   346.164        NA
## ...       ...       ...       ...
## 58          2   551.735        NA
## 59          2   551.735        NA
## 60          2   575.134        NA
## 61          2   575.134        NA
## 62          2   574.942        NA
##  ... 20 more variables/columns.
## Processing:
##  Merge 1 Spectra into one [Tue Apr 26 18:26:01 2022]

As a result we got a Spectra object of length equal to the number of MS1 peaks in our data. A peaksCount of 0 indicates that no MS2 spectrum could be defined based on the used settings. For reconstructed spectra additional annotations are available such as the IDs of the MS2 chromatographic peaks from which the spectrum was reconstructed ("ms2_peak_id") as well as the correlation coefficient of their chromatographic peak shape with the precursor’s shape ("ms2_peak_cor"). Metadata column "peak_id" contains the ID of the MS1 chromatographic peak:

> 因此，我们得到了一个长度等于数据中MS1峰数的光谱对象。峰值计数为0表示无法根据所用设置定义MS2频谱。对于重建的光谱，可以使用其他注释，例如重建光谱的MS2色谱峰的id（“MS2\u peak\u id”）以及其色谱峰形状与前体形状的相关系数（“MS2\u peak\u cor”）。元数据列“peak_id”包含MS1色谱峰的id：

swath_spectra$ms2_peak_id
## CharacterList of length 62
## [[1]] character(0)
## [[2]] character(0)
## [[3]] CP063
## [[4]] CP105
## [[5]] CP153
## [[6]] character(0)
## [[7]] character(0)
## [[8]] character(0)
## [[9]] character(0)
## [[10]] character(0)
## ...
## <52 more elements>
swath_spectra$peak_id
##  [1] "CP01" "CP02" "CP03" "CP04" "CP05" "CP06" "CP07" "CP08" "CP09" "CP10"
## [11] "CP11" "CP12" "CP13" "CP14" "CP15" "CP16" "CP17" "CP18" "CP19" "CP20"
## [21] "CP21" "CP22" "CP23" "CP24" "CP25" "CP26" "CP27" "CP28" "CP29" "CP30"
## [31] "CP31" "CP32" "CP33" "CP34" "CP35" "CP36" "CP37" "CP38" "CP39" "CP40"
## [41] "CP41" "CP42" "CP43" "CP44" "CP45" "CP46" "CP47" "CP48" "CP49" "CP50"
## [51] "CP51" "CP52" "CP53" "CP54" "CP55" "CP56" "CP57" "CP58" "CP59" "CP60"
## [61] "CP61" "CP62"

We next extract the MS2 spectrum for our example peak most likely representing [M+H]+ ions of Fenamiphos using its chromatographic peak ID:

fenamiphos_swath_spectrum <- swath_spectra[
    swath_spectra$peak_id == rownames(fenamiphos_ms1_peak)]
    
We can now compare the reconstructed spectrum to the example consensus spectrum from the DDA experiment in the previous section (variable ex_spectrum) as well as to the MS2 spectrum for Fenamiphos from Metlin (with a collision energy of 10V). For better visualization we normalize also the peak intensities of the reconstructed SWATH spectrum with the same function we used for the experimental DDA spectrum.

> 现在，我们可以将重建的光谱与上一节DDA实验中的示例一致光谱（可变ex_光谱）以及Metlin的非那米磷MS2光谱（碰撞能量为10V）进行比较。为了更好地可视化，我们还使用与实验DDA光谱相同的函数对重建的条带光谱的峰值强度进行归一化。

fenamiphos_swath_spectrum <- addProcessing(fenamiphos_swath_spectrum,
                                           norm_fun)
par(mfrow = c(1, 2))
plotSpectraMirror(fenamiphos_swath_spectrum, ex_spectrum,
     ppm = 50, main = "against DDA")
plotSpectraMirror(fenamiphos_swath_spectrum, fenamiphos[2],
     ppm = 50, main = "against Metlin")
     
Mirror plot comparing the reconstructed MS2 spectrum for Fenamiphos (upper panel) against the measured spectrum from the DDA data and the Fenamiphhos spectrum from Metlin.
Figure 4: Mirror plot comparing the reconstructed MS2 spectrum for Fenamiphos (upper panel) against the measured spectrum from the DDA data and the Fenamiphhos spectrum from Metlin
If we wanted to get the EICs for the MS2 chromatographic peaks used to generate this MS2 spectrum we can use the IDs of these peaks which are provided with $ms2_peak_id of the result spectrum.

> 镜像图，将重建的非那米磷MS2光谱（上面板）与DDA数据的测量光谱和Metlin的非那米磷光谱进行比较。
图4：镜像图，将非那米磷重建的MS2光谱（上面板）与DDA数据的测量光谱和Metlin的非那米磷光谱进行比较
如果我们想要获得用于生成该MS2光谱的MS2色谱峰的EIC，我们可以使用这些峰的id，这些峰的id随结果光谱的$MS2\u peak\u id提供。

pk_ids <- fenamiphos_swath_spectrum$ms2_peak_id[[1]]
pk_ids
##  [1] "CP199" "CP201" "CP211" "CP208" "CP200" "CP202" "CP217" "CP215" "CP205"
## [10] "CP212" "CP221" "CP223" "CP213" "CP207" "CP220"


With these peak IDs available we can extract their retention time window and m/z ranges from the chromPeaks matrix and use the chromatogram function to extract their EIC. Note however that for SWATH data we have MS2 signal from different isolation windows. Thus we have to first filter the swath_data object by the isolation window containing the precursor m/z with the filterIsolationWindow to subset the data to MS2 spectra related to the ion of interest. In addition, we have to use msLevel = 2L in the chromatogram call because chromatogram extracts by default only data from MS1 spectra.

> 有了这些峰ID，我们可以从色谱峰矩阵中提取其保留时间窗口和m/z范围，并使用色谱功能提取其EIC。然而，请注意，对于小水线带数据，我们有来自不同隔离窗口的MS2信号。因此，我们必须首先通过包含前体m/z的隔离窗口和filterIsolationWindow对swath_数据对象进行过滤，以将数据子集到与感兴趣离子相关的MS2光谱。此外，我们必须在色谱调用中使用msLevel=2L，因为色谱默认仅从MS1光谱中提取数据。

rt_range <- chromPeaks(swath_data)[pk_ids, c("rtmin", "rtmax")]
mz_range <- chromPeaks(swath_data)[pk_ids, c("mzmin", "mzmax")]

pmz <- precursorMz(fenamiphos_swath_spectrum)[1]
swath_data_iwindow <- filterIsolationWindow(swath_data, mz = pmz)
## Warning in object[keep]: Removed preprocessing results
ms2_eics <- chromatogram(swath_data_iwindow, rt = rt_range,
                         mz = mz_range, msLevel = 2L)
                         
Each row of this ms2_eics contains now the EIC of one of the MS2 chromatographic peaks.

As a second example we analyze the signal from an [M+H]+ ion with an m/z of 376.0381 (which would match Prochloraz). We first identify the MS1 chromatographic peak for that m/z and retrieve the reconstructed MS2 spectrum for that peak.

> 该ms2_EIC的每一行现在包含一个ms2色谱峰的EIC。
作为第二个例子，我们分析了M/z为376.0381（与咪鲜胺匹配）的[M+H]+离子的信号。我们首先确定该m/z的MS1色谱峰，并检索该峰的重建MS2光谱。

prochloraz_mz <- 376.0381

prochloraz_ms1_peak <- chromPeaks(swath_data, msLevel = 1L,
                                  mz = prochloraz_mz, ppm = 5)
prochloraz_ms1_peak
##            mz   mzmin    mzmax      rt   rtmin   rtmax     into     intb
## CP22 376.0373 376.037 376.0374 405.046 401.446 409.546 3664.051 3655.951
##          maxo  sn sample
## CP22 897.3923 278      1
prochloraz_swath_spectrum <- swath_spectra[
    swath_spectra$peak_id == rownames(prochloraz_ms1_peak)]
In addition we identify the corresponding MS1 peak in the DDA data set, extract all measured MS2 chromatographic peaks and build the consensus spectrum from these.

prochloraz_dda_peak <- chromPeaks(dda_data, msLevel = 1L,
                                  mz = prochloraz_mz, ppm = 5)
prochloraz_dda_peak
##             mz    mzmin    mzmax      rt   rtmin   rtmax     into     intb
## CP034 376.0385 376.0378 376.0391 405.715 400.346 410.555 5088.528 5078.727
##           maxo  sn sample
## CP034 1350.633 436      1

The retention times for the chromatographic peaks from the DDA and SWATH data match almost perfectly. Next we get the MS2 spectra for this peak.

> DDA和SWATH数据中色谱峰的保留时间几乎完全匹配。接下来我们得到这个峰的MS2光谱。

prochloraz_dda_spectra <- dda_spectra[
    dda_spectra$peak_id == rownames(prochloraz_dda_peak)]
prochloraz_dda_spectra
## MSn data (Spectra) with 5 spectra in a MsBackendMzR backend:
##            msLevel     rtime scanIndex
##          <integer> <numeric> <integer>
## F1.S3253         2   401.438      3253
## F1.S3259         2   402.198      3259
## F1.S3306         2   404.677      3306
## F1.S3316         2   405.127      3316
## F1.S3325         2   405.877      3325
##  ... 38 more variables/columns.
## 
## file(s):
## PestMix1_DDA.mzML

In total 5 spectra were measured, some with a relatively high number of peaks. Next we combine them into a consensus spectrum.

> 共测量了5个光谱，其中一些具有相对较高的峰数。接下来，我们将它们组合成一个共识谱。

prochloraz_dda_spectrum <- combineSpectra(
    prochloraz_dda_spectra, FUN = combinePeaks, ppm = 20,
    peaks = "intersect", minProp = 0.8, intensityFun = median, mzFun = median,
    f = prochloraz_dda_spectra$peak_id)
## Backend of the input object is read-only, will change that to an 'MsBackendDataFrame'
## At last we load also the Prochloraz MS2 spectra (for different collision energies) from Metlin.

prochloraz <- Spectra(
    system.file("mgf", "metlin-68898.mgf", package = "xcms"),
    source = MsBackendMgf())
## Start data import from 1 files ... done

To validate the reconstructed spectrum we plot it against the corresponding DDA spectrum and the MS2 spectrum for Prochloraz (for a collision energy of 10V) from Metlin.

> 为了验证重建的光谱，我们将其与Metlin的相应DDA光谱和咪鲜胺的MS2光谱（碰撞能量为10V）进行了对比。

prochloraz_swath_spectrum <- addProcessing(prochloraz_swath_spectrum, norm_fun)
prochloraz_dda_spectrum <- addProcessing(prochloraz_dda_spectrum, norm_fun)

par(mfrow = c(1, 2))
plotSpectraMirror(prochloraz_swath_spectrum, prochloraz_dda_spectrum,
                  ppm = 40, main = "against DDA")
plotSpectraMirror(prochloraz_swath_spectrum, prochloraz[2],
                  ppm = 40, main = "against Metlin")
                  
Mirror plot comparing the reconstructed MS2 spectrum for Prochloraz (upper panel) against the measured spectrum from the DDA data and the Prochloraz spectrum from Metlin.

The spectra fit relatively well. Interestingly, the peak representing the precursor (the right-most peak) seems to have a slightly shifted m/z value in the reconstructed spectrum.

Similar to the DDA data, the reconstructed MS2 spectra from SWATH data could be used in the annotation of the MS1 chromatographic peaks.

> 镜像图，将咪鲜胺（上面板）的重建MS2光谱与DDA数据的测量光谱和Metlin的咪鲜胺光谱进行比较。
光谱拟合相对较好。有趣的是，代表前体的峰（最右边的峰）在重建的光谱中似乎有轻微的移动m/z值。
与DDA数据类似，从SWATH数据重建的MS2光谱可用于注释MS1色谱峰。

## 4Outlook
Currently, spectra data representation, handling and processing is being re-implemented as part of the RforMassSpectrometry initiative aiming at increasing the performance of methods and simplifying their use. Thus, parts of the workflow described here will be changed (improved) in future.

> 目前，光谱数据表示、处理和处理正在重新实施，作为rformasspectronics倡议的一部分，旨在提高方法的性能并简化其使用。因此，这里描述的部分工作流程将在将来进行更改（改进）。

Along with these developments, improved matching strategies for larger data sets will be implemented as well as functionality to compare Spectra directly to reference MS2 spectra from public annotation resources (e.g. Massbank or HMDB). See for example here for more information.

> 随着这些发展，将实施更大数据集的改进匹配策略，以及直接将光谱与公共注释资源（例如Massbank或HMDB）中的参考MS2光谱进行比较的功能。有关更多信息，请参阅此处的示例。

Regarding SWATH data analysis, future development will involve improved selection of the correct MS2 chromatographic peaks considering also correlation with intensity values across several samples.

> 关于SWATH数据分析，未来的发展将涉及改进正确MS2色谱峰的选择，同时考虑到与多个样本的强度值的相关性。

## References
1. Ludwig C, Gillet L, Rosenberger G, Amon S, Collins BC, Aebersold R: Data-independent acquisition-based SWATH-MS for quantitative proteomics: a tutorial. Molecular systems biology 2018, 14:e8126.


