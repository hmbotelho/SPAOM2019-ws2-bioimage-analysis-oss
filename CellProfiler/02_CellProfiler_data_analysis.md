# SPAOM 2019<br>WS2: Bioimage analysis open-source software tools

## Data analysis with R

![R Jupyter logo](../misc/r_jupyter_logo1.png)


This notebook performs basic statistical analyses on the image quantification data obtained from CellProfiler.

**Note:** This notebook should run on any browser but some interactive features will only be available on Firefox.

## Table of Contents
* [1. Foreword](#foreword)  
* [2. The Goal](#goal)  
* [3. [Statistics](#statistics)  
* [4. Image analysis performance](#performance)  
* [5. Data Analysis](#analysis)  
  * [5.1. Load Data](#loaddata)  
  * [5.2. Data preprocessing](#preprocessing)  
  * [5.3. Data summary and normalization](#summary)  
  * [5.4. Plotting results](#plotting)  
* [6. Results interpretation](#interpretation)  


## <a name="foreword">1. Foreword</a>
This notebook is part of an analysis workflow for the Forskolin-induced Swelling (FIS) assay (more information [here](https://github.com/hmbotelho/SPAOM2019-ws2-bioimage-analysis-oss/blob/master/README.md)), as depicted in the figure below:

![Analysis workflow](../misc/workflow.gif)

This notebook requires that:
* Raw microscopy images have been analyzed with CellProfiler in order to extract:
    * Time-resolved per-object cross-sectional area
    * Experimental metadata (time, experimental treatment, ...)
* Object features are available on a CSV file

CellProfiler is a powerful tool for image processing and feature extraction, both at the image- and object-levels. This means that it is frequently coupled with statistical analysis environments (like those available in R or Python).


## <a name="goal">2. The Goal</a>
The goal of this analysis is to determine the amount of swelling in each experimental condition:
1. Mutant organoids
2. Mutant organoids + CFTR inhibitor
3. Genome edited organoids (clone 1)
4. Genome edited organoids (clone 2)
5. Genome edited organoids (clone 1) + CFTR inhibitor
6. Genome edited organoids (clone 2) + CFTR inhibitor

and determining which genome/edited clone better rescues the CFTR mutation (*i.e.* produces the most swelling).

Achieving this goal will require:
* Summarizing the CellProfiler output (one measurement per time point per condition)
* Reshaping the data table
* Normalizing results
* Plotting


## <a name="statistics">3. Statistics</a>
CellProfiler allows extracting **object-level image features**. This means that the `objects.csv` file we obtain after running CellProfiler is a tabular dataset, where each line corresponds to the measurements performed on 1 object located in 1 image. In our case, we will be interested in the cross-sectional area, measured in μm² (column `Math_area_micronsq`). 
Such data representation is desirable because one can refer back to the object which generated that piece of data, and - if required - perform **object-level quality control**. On the other hand, this data representation requires **further processing** in order to obtain **summarized statistics** (*e.g.* one value per image or experimental condition).
To measure organoid swelling we will follow the following algorithm:
1. Read the `objects.csv` file
2. Compute the sum of the area of all organoids on each image
3. Normalize each kinetics to the initial time point (Area @ t₀ = 100%)
4. Compute the area under the normalized kinetics curve (AUC), using 100% as baseline, as shown in the following scheme:
![AUC scheme](https://raw.githubusercontent.com/hmbotelho/SPAOM2019/master/misc/AUC_scheme.png)
5. Plot the summarized data and the AUC at the final experiment time (55 min).

We will also examine the CellProfiler segmentation performance, to validate the results.


Let us load a few useful libraries and functions:


```R
# Load dependencies
library(dplyr)
library(caTools)
library(plotly)
library(ggplot2)
library(magick)

# Calculates the AUC for increasingly larger curves: [x0,y0 : x0,y0], [x0,y0 : x1,y1], [x0,y0 : x2,y2], [x0,y0 : x3,y3], ...
cumulativeAUC <- function(x, y){
    # library(caTools)
    
    cumulateInt <- numeric(length(x))
    names(cumulateInt) <- x
    if(is.na(y[1])){
        cumulateInt[1] <- NA
    } else{
        cumulateInt[1] <- 0 
    }
    
    for (i in 2:length(x)){
        # Integration range
        IntRange <- 1:i
        # Function trapz belongs to package "caTools"
        cumulateInt[i] <- trapz(x[IntRange], y[IntRange])
    }
    cumulateInt
}



# Overlays and shows a smooth transition 
# between all raw images and object labels in an arbitrary well
checkSegmentation <- function(wellNum, folderRawImg, folderLabelImg){
    # library(magick)
    
    # Names of all images: raw and labels
    all.images.raw <- paste0(rep("/W00", 72),
                             rep(1:6, each=12),
                             rep("--", 72),
                             rep(rep(c("mutant","corrected1","corrected2"), each=12),2),
                             rep("--", 72),
                             rep(c("control", "inhibitor"), each=36),
                             rep("/W00", 72),
                             rep(1:6, each=12),
                             rep("--", 72),
                             rep(rep(c("mutant","corrected1","corrected2"), each=12),2),
                             rep("--", 72),
                             rep(c("control", "inhibitor"), each=36),
                             rep("--", 72),
                             formatC(seq(0,55,5), width=2, flag="0"),
                             rep("min.tif", 72))
    all.images.labels <- gsub("min.tif", "min--labels_tracked.tiff", all.images.raw)
    
    
    # Get indices of all images in the requested well
    indices <- which(grepl(paste0("W00", wellNum), all.images.raw))
    
    
    # Make montage with raw images
    if(exists("imgs_raw")) rm(imgs_raw)
    for(i in indices){
        pathRaw <- paste0(folderRawImg, all.images.raw[i])
        img_raw <- image_read(pathRaw) %>%
            image_modulate(brightness = 60000) %>%
            image_annotate(sub(".*(..min).*","\\1",pathRaw), size=50, color="white", gravity="NorthEast")
        if(!exists("imgs_raw")){
            imgs_raw <- img_raw
        } else{
            imgs_raw <- c(imgs_raw, img_raw)
        }
    }
    montage_raw <- image_montage(imgs_raw)
    
    
    # Make montage with labels images
    if(exists("imgs_labels")) rm(imgs_labels)
    for(i in indices){
        pathLabels <- paste0(folderLabelImg, all.images.labels[i])
        img_labels <- image_read(pathLabels) %>%
            image_annotate(sub(".*(..min).*","\\1",pathLabels), size=50, color="white", gravity="NorthEast")
        if(!exists("imgs_labels")){
            imgs_labels <- img_labels
        } else{
            imgs_labels <- c(imgs_labels, img_labels)
        }
    }
    montage_labels <- image_montage(imgs_labels)
    
    
    # Animate between raw & labels
    frames <- image_morph(c(montage_raw, montage_labels), frames = 10)
    
    
    # Display results
    print(paste0("Showing segmentation for well ", wellNum, ":"), quote=F)
    image_animate(frames)
}
```

    
    Attaching package: 'dplyr'
    
    The following objects are masked from 'package:stats':
    
        filter, lag
    
    The following objects are masked from 'package:base':
    
        intersect, setdiff, setequal, union
    
    Loading required package: ggplot2
    Registered S3 methods overwritten by 'ggplot2':
      method         from 
      [.quosures     rlang
      c.quosures     rlang
      print.quosures rlang
    
    Attaching package: 'plotly'
    
    The following object is masked from 'package:ggplot2':
    
        last_plot
    
    The following object is masked from 'package:stats':
    
        filter
    
    The following object is masked from 'package:graphics':
    
        layout
    
    Linking to ImageMagick 6.9.9.14
    Enabled features: cairo, freetype, fftw, ghostscript, lcms, pango, rsvg, webp
    Disabled features: fontconfig, x11
    

### <a performance="qc">4. Image analysis performance</a>

Let us check whether segmentation worked.
This function may take several seconds to run. Please be patient.


```R
checkSegmentation(wellNum        = 2, 
                  folderRawImg   = "https://github.com/hmbotelho/SPAOM2019-ws2-bioimage-analysis-oss/raw/master/images",
                  folderLabelImg = "https://github.com/hmbotelho/SPAOM2019-ws2-bioimage-analysis-oss/raw/master/CellProfiler/analysis--cellprofiler")
```

    [1] Showing segmentation for well 2:
    

![Segmentation results](misc/segmentationwell2.gif)



Looks like segmentation worked very well. We can now continue with the statstical analysis.

## <a name="analysis">5. Data Analysis</a>

### <a name="loaddata">5.1. Load Data</a>
Let us start by loading the data


```R
objectsfile <- "https://raw.githubusercontent.com/hmbotelho/SPAOM2019-ws2-bioimage-analysis-oss/master/CellProfiler/analysis--cellprofiler/objects.csv"
data <- read.csv(objectsfile)
```

### <a name="preprocessing">5.2. Data preprocessing</a>

Let us rearrange the data.


```R
# rename columns
colnames(data) <- gsub("Metadata_", "", names(data))
data$treatment <- paste0(data$genotype, "_", data$assay)
mylevels <- c("corrected1_control",
              "corrected2_control",
              "mutant_control",
              "mutant_inhibitor",
              "corrected1_inhibitor",
              "corrected2_inhibitor")
data$treatment <- factor(data$treatment, levels = mylevels)
```

Let us see how the data looks like:


```R
head(data[,c(2,6:9,11,13)])
```


<table>
<thead><tr><th scope=col>ObjectNumber</th><th scope=col>assay</th><th scope=col>genotype</th><th scope=col>minutes</th><th scope=col>wellNum</th><th scope=col>Math_area_micronsq</th><th scope=col>treatment</th></tr></thead>
<tbody>
	<tr><td>1             </td><td>control       </td><td>mutant        </td><td>0             </td><td>1             </td><td> 8867.25      </td><td>mutant_control</td></tr>
	<tr><td>2             </td><td>control       </td><td>mutant        </td><td>0             </td><td>1             </td><td>32393.25      </td><td>mutant_control</td></tr>
	<tr><td>3             </td><td>control       </td><td>mutant        </td><td>0             </td><td>1             </td><td>25377.75      </td><td>mutant_control</td></tr>
	<tr><td>4             </td><td>control       </td><td>mutant        </td><td>0             </td><td>1             </td><td>12606.75      </td><td>mutant_control</td></tr>
	<tr><td>5             </td><td>control       </td><td>mutant        </td><td>0             </td><td>1             </td><td>22189.50      </td><td>mutant_control</td></tr>
	<tr><td>6             </td><td>control       </td><td>mutant        </td><td>0             </td><td>1             </td><td>12235.50      </td><td>mutant_control</td></tr>
</tbody>
</table>



At this point we could perform additional quality control:
* **Image-based quality control**: exclude images with aberrations (*e.g.* out-of focus, no organoids)
* **Object-based quality control:** exclude individual organoids based on their features (*e.g.* dead/disrupted organoids).

We will not be performing any of these and will continue with the dataset as it stands.


### <a name="summary">5.3. Data summary and normalization</a>

Let us compute summary statistics:
* Sum organoid area in each image
* Normalize each kinetics curve to the initial data point (area at time 0 = 100%)
* Compute the area under the normalized curve (baseline: 100%)


```R
normdata <- data %>%
    group_by(wellNum, minutes) %>%                              # Split data by well
    summarise(genotype   = unique(genotype),                    # Add metadata
              assay      = unique(assay),                       # Add metadata
              treatment  = unique(treatment),                   # Add metadata
              sumarea    = sum(Math_area_micronsq)) %>%         # Sum area of all objects
    mutate(normalized    = sumarea / sumarea[1] * 100,          # Normalize (sumarea @ 0min = 100%)
           offset        = normalized - 100,                    # Offset (sumarea @ 0min = 0)
           AUC           = cumulativeAUC(minutes, offset))      # AUC, cumulative

head(normdata)
```


<table>
<thead><tr><th scope=col>wellNum</th><th scope=col>minutes</th><th scope=col>genotype</th><th scope=col>assay</th><th scope=col>treatment</th><th scope=col>sumarea</th><th scope=col>normalized</th><th scope=col>offset</th><th scope=col>AUC</th></tr></thead>
<tbody>
	<tr><td>1             </td><td> 0            </td><td>mutant        </td><td>control       </td><td>mutant_control</td><td>147654.0      </td><td>100.00000     </td><td> 0.00000000   </td><td>0.000000      </td></tr>
	<tr><td>1             </td><td> 5            </td><td>mutant        </td><td>control       </td><td>mutant_control</td><td>149827.5      </td><td>101.47202     </td><td> 1.47202243   </td><td>3.680056      </td></tr>
	<tr><td>1             </td><td>10            </td><td>mutant        </td><td>control       </td><td>mutant_control</td><td>146290.5      </td><td> 99.07656     </td><td>-0.92344264   </td><td>5.051506      </td></tr>
	<tr><td>1             </td><td>15            </td><td>mutant        </td><td>control       </td><td>mutant_control</td><td>147687.8      </td><td>100.02286     </td><td> 0.02285749   </td><td>2.800043      </td></tr>
	<tr><td>1             </td><td>20            </td><td>mutant        </td><td>control       </td><td>mutant_control</td><td>147768.8      </td><td>100.07772     </td><td> 0.07771547   </td><td>3.051475      </td></tr>
	<tr><td>1             </td><td>25            </td><td>mutant        </td><td>control       </td><td>mutant_control</td><td>149211.0      </td><td>101.05449     </td><td> 1.05449226   </td><td>5.881994      </td></tr>
</tbody>
</table>



This is a grphical representation of the AUC. The idea is that the AUC is a measure of overall swelling.

![AUC scheme](../misc/AUC_scheme1.png)

### <a name="plotting">5.4. Plotting results</a>

The following plots (`plotly`) cannot be generated in Chrome.


```R
# Kinetics: % swelling
p1 <- plot_ly(data = normdata, 
              x = ~minutes, 
              y = ~normalized, 
              type = "scatter",
              mode = 'lines+markers',
              text = paste("genotype: ", normdata$genotype,
                           "<br>assay: ", normdata$assay,
                           "<br>time: ", normdata$minutes, " min",
                           "<br>cumulative AUC: ", round(normdata$AUC,1)),
              hoverinfo = 'text',
              color = ~treatment) %>%
      layout(xaxis = list(title = "Time (min)"),
             yaxis = list (title = "Sum area (%)"))

# Kinetics: AUC (area under the curve)
p2 <- plot_ly(data = normdata, 
              x = ~minutes, 
              y = ~AUC, 
              type = "scatter",
              mode = 'lines+markers',
              text = paste("genotype: ", normdata$genotype,
                           "<br>assay: ", normdata$assay,
                           "<br>time: ", normdata$minutes, " min",
                           "<br>cumulative AUC: ", round(normdata$AUC,1)),
              hoverinfo = 'text',
              color = ~treatment) %>%
      layout(xaxis = list(title = "Time (min)"),
             yaxis = list (title = "Cumulative AUC"))

## Kinetics: summary at 55 min
normdata55 <- subset(normdata, minutes == 55)
p3 <- plot_ly(data = normdata55, 
              x = ~treatment, 
              y = ~AUC, 
              type = "bar",
              text = paste("genotype: ", normdata55$genotype,
                           "<br>assay: ", normdata55$assay,
                           "<br>time: ", normdata55$minutes, " min",
                           "<br>cumulative AUC: ", round(normdata55$AUC,1)),
              hoverinfo = 'text') %>%
      layout(xaxis = list(title = "Treatment"),
             yaxis = list (title = "Cumulative AUC at 55 min"))

embed_notebook(p1)
embed_notebook(p2)
embed_notebook(p3)
```



![plotly plots](misc/plotly.png)




The following plots (`ggplot2`) can be generated in all browsers.


```R
# Kinetics: % swelling
g1 <- ggplot(data = normdata, aes(minutes, normalized, color = treatment)) + 
      geom_line(size=1) + 
      geom_point(size=3) + 
      xlab ("Time (min)") +
      ylab ("Sum area (%)") + 
      geom_hline(yintercept = 100, size=1)

# Kinetics: AUC (area under the curve)
g2 <- ggplot(data = normdata, aes(minutes, AUC, color = treatment)) + 
      geom_line(size=1) + 
      geom_point(size=3) + 
      xlab ("Time (min)") +
      ylab ("Cumulative AUC") + 
      geom_hline(yintercept = 0, size=1)

## Kinetics: summary at 55 min
normdata55 <- subset(normdata, minutes == 55)
g3 <- ggplot(data = normdata55, aes(treatment, AUC)) + 
      geom_bar(stat="identity") +
      xlab ("Treatment") +
      ylab ("Cumulative AUC at 55 min") + 
      theme(axis.text.x = element_text(angle = 45, hjust = 1)) +
      geom_hline(yintercept = 0, size=1)

g1
g2
g3
```


![png](misc/output_15_0.png)



![png](misc/output_15_1.png)



![png](misc/output_15_2.png)


## <a name="interpretation">6. Results interpretation</a>

Clone #1 is the best at correcting the CFTR mutation, as it produces the most swelling. We can be sure that this clone is indeed correcting CFTR because the usage of a specific CFTR inhibitor completely inhibits swelling.
