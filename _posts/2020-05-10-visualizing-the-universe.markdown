---
layout: post
title:  "Visualizing the Universe with VTK(Part 1)"
date:   2020-05-10 10:00:00 -0600
categories: 10 Min Read
---
Last semester, I started worked with VTK (The Visualization Toolkit) for a course (CS530) and learnt some cool visualization methods. VTK makes visualization very simple by using a pipeline approach. Most VTK codes can be broadly broken down to 5 components. 

* Source: This is the data itself. Vtk data sets usually end with .vti or .vtk etc. The source is read using vtk reader for eg vtkXMLImageDataReader, vtkStructuredGridReader etc
* Filter: Here you can modify the data as wish. VtkContourFilter is used to find iso-surfaces, vtkClipPolyData to remove a slice of the data etc
* Mapper: The data output of filter cannot be directly rendered. Mapper maps the output of the filter to graphical primitives (triangles, cubes etc) that can then be rendered. 
* Actor: It defines the properties, such as texture,color etc of the object being rendered
* Renderer: Puts everything together and renders/creates the objects. Also controls the camera lighting etc of the rendered objects. 

Usually these steps follow sequentially. The output one goes as input to the next one. However, it is not mandatory. We can skip the source and filter if need be. The last 3 steps are essential though. In our case we will skip the filter.

First let’s set everything up. You will need the following packages in python

* Numpy 
* H5py to convert hdf data numpy arrays 
* Vtk

You can get the complete code [here](https://github.com/anirban1195/vtk/blob/master/darkmatterdensity_dvr.py)

## The Dataset
The dataset is a MHD simulation of a galaxy cluster by F.Vazza et al [available here](https://b2share.eudat.eu/records/6def51ee4fc943f8915994fcc9c82ec7). We will use "snap_s138.hdf". First lets have a quick look at the data.
{% highlight python %}
filename = 'snap_s138.hdf' 
with h5py.File(filename, 'r') as f: 
    # List all groups 
    print("Keys: %s" % f.keys()) 

{% endhighlight %}

This gives

{% highlight python %}
Keys: <KeysViewHDF5 ['Bx', 'By', 'Bz', 'Dark_Matter_Density', 'Density', 'Temperature']> 
{% endhighlight %}

Let's extract the Dark Matter Densityand get a feel of the data. For now, we will not bother about the units. 

{% highlight python %}
with h5py.File(filename, 'r') as f: 
    idx = list(f.keys())[3] 
    # Get the data 
    data = list(f[idx]) 
data=np.array(data) 
print (data.shape) 
{% endhighlight %}

We see it is a 320x320x320 array.Each edge of the data cube corresponds to 7Mpc. So, each small cube edge corresponds to about 21.9kpc, or about half the size of Milky Way. While we are at it, let’s also try to determine distribution of values by plotting histogram.

{% highlight python %}
n, bins, patches = plt.hist(x=data.flatten(), bins=100, alpha=0.7, rwidth=0.85) 
{% endhighlight %}
![Histogram 1](/images/vtkImg/hist_0.png)

Clearly most of the data is in the first bin. Let's clip the data up to the first bin(about 100) and plot again.

{% highlight python %}
n, bins, patches = plt.hist(x=data[data<100].flatten(), bins=100, alpha=0.7, rwidth=0.85) 
{% endhighlight %}
![Histogram 2](/images/vtkImg/hist_1.png)


We again end up with a similar looking histogram. If we continue this pattern, we find most of the values are close to zero. Let us also find the maximum and minimum values of the array.

{% highlight python %}
print ('Max ='+str(np.max(data))) 
print ('Min =' +str(np.min(data)))
 
{% endhighlight %}
Gives
{% highlight python %}
Max =14810.714
Min =0.0004907315
{% endhighlight %}
So, in the data spans almost 8 orders of magnitude, with most values less than 0.1. This might not seem to be a problem but for now you will just have to trust me when I say such huge range of data is bad news. The most obvious solution is to take the log. To avoid the issue of taking log of zeros I add 1 to the entire array. There is nothing special about 1.

{% highlight python %}
data = data + 1
data = np.log10(data) 
{% endhighlight %}

This should be enough preprocessing. We can now move on the VTK part.

## Importing to VTK  

We need to import the 3-D numpy array as a VTK type. There are several VTK data types such as Unstructured Grid, Structured Grid, Image (uniform grid) etc. Since we have a uniform grid, we convert the numpy array to VTK image.This part of the code roughly corresponds to source. 

{% highlight python %}
dataImporter = vtk.vtkImageImport()
data_string = data.tostring()
dataImporter.CopyImportVoidPointer(data_string, len(data_string))
dataImporter.SetDataScalarTypeToFloat()
dataImporter.SetNumberOfScalarComponents(1)
dataImporter.SetDataExtent(0, 319, 0, 319, 0, 319)
dataImporter.SetWholeExtent(0, 319, 0, 319, 0, 319)
{% endhighlight %}

We set the data type as float in line 4 and the number of scalar components to 1 in line 6 since we have only one number associated with each point in space. The last 2 lines set the extent of the data which in our case is 320x320x320. 

## Lookup Table and Opacity Function

At this point what we have is a uniform 3-D grid, with each voxel having some value. To represent this, we need to map each value to a color. You are probably familiar with the fact that colors are often associated with values. A classical physics example is blue corresponds to cool color (ironically from a hotter black blackbody ) whereas shades of red represent warmer colors (corresponding to a relatively lower blackbody temperature).
We need to explicitly tell vtk which values correspond to what color. This is called a color transfer function.
To tell vtk we want voxels with  value 4.0 displayed red, simply say  colorTransferFunction.AddRGBPoint(4,1,0,0). The last 3 numbers correspond to RGB values respectively and varies between 0 and 1.
For this case lets go with a color scheme from red to yellow to blue, with red corresponding to the highest value and blue the lowest. The code for this is.

{% highlight python %}
colorTransferFunction = vtk.vtkColorTransferFunction()
colorTransferFunction.AddRGBPoint(0,0,0,1)
colorTransferFunction.AddRGBPoint(2,1,1,0)
colorTransferFunction.AddRGBPoint(4,1,0,0)
{% endhighlight %}

First 2 lines create a color transfer function with color space set to RGB. Then we simply add the necessary points. For values between these points, the colors are linearly interpolated by vtk.
Similarly, we need to define an opacity transfer function as well. As the name suggests, this function maps values to opacity. We need this if we want to get a sense of structure. If each voxel is completely opaque or completely transparent, it does not give us much information.  Determining a good opacity transfer function is not a trivial problem. Hit and trial method works well when you have an idea of the data being used.Just like the previous case we set the opacity at various values. 

{% highlight python %}
opacityTransferFunction = vtk.vtkPiecewiseFunction()
opacityTransferFunction.AddPoint(0, 0.0)

opacityTransferFunction.AddPoint(1, 0.0)
opacityTransferFunction.AddPoint(1.01, 0.2)
opacityTransferFunction.AddPoint(1.15, 0.2)
opacityTransferFunction.AddPoint(1.16 ,0.0)

opacityTransferFunction.AddPoint(1.70, 0.0)
opacityTransferFunction.AddPoint(1.71, 0.2)
opacityTransferFunction.AddPoint(1.85, 0.2)
opacityTransferFunction.AddPoint(1.86 ,0.0)

opacityTransferFunction.AddPoint(2, 0.0)
opacityTransferFunction.AddPoint(2.01, 0.2)
opacityTransferFunction.AddPoint(2.15, 0.2)
opacityTransferFunction.AddPoint(2.16 ,0.0)

opacityTransferFunction.AddPoint(2.71, 0.0)
opacityTransferFunction.AddPoint(2.71, 0.2)
opacityTransferFunction.AddPoint(2.85, 0.2)
opacityTransferFunction.AddPoint(2.86 ,0.0)

opacityTransferFunction.AddPoint(3, 0.0)
opacityTransferFunction.AddPoint(3.01, 0.2)
opacityTransferFunction.AddPoint(3.85, 0.2)
opacityTransferFunction.AddPoint(3.86 ,0.0)

opacityTransferFunction.AddPoint(4, 0.0)
opacityTransferFunction.AddPoint(4.01, 0.2)
opacityTransferFunction.AddPoint(4.85, 0.2)
opacityTransferFunction.AddPoint(4.86 ,0.0)
{% endhighlight %}

The opacity and color transfer function together is called transfer function. We represent it as shown below. On y axis you see opacity and on x axis we have the values. We get the color at each point from the color transfer function.
![Transfer Function](/images/vtkImg/dark_matter_tf.png)

## Volume Actor and Mapper

We also need to specify the property of the volume in which our picture will be displayed and the volume mapper. We use the following lines of code.

{% highlight python %}
# The property describes how the data will look
volumeProperty = vtk.vtkVolumeProperty()
volumeProperty.SetColor(colorTransferFunction)
volumeProperty.SetScalarOpacity(opacityTransferFunction)
volumeProperty.SetScalarOpacityUnitDistance(10)
volumeProperty.ShadeOn()
volumeProperty.SetInterpolationTypeToLinear()

# The mapper / ray cast function know how to render the data
volumeMapper = vtk.vtkSmartVolumeMapper()
volumeMapper.SetBlendModeToComposite()
volumeMapper.SetInputConnection(dataImporter.GetOutputPort())
{% endhighlight %}

The code is self explanatory. The 4th  line tells vtk to consider 10 voxel as unit distance.Just opacity is not enough to calculate proper amount of light transmission. We also need how thick the objects are. Setting unit distance helps us do that.At this point it is worth pointing out the pipeline nature of vtk. The last line in the last code snippet is a typical example. It simply takes the output of dataImporter and sets it as an input for volumeMapper. Think of this as adding the object in the virtual space we have created with. The virtual space has properties that are defined by the transfer functions, unit distance, shading etc.

Vtk smart mapper is something I use by default. It tells vtk to use GPU if available else use CPU. Basically, doing the smart stuff to speed up things. Now all we need to do is add these lines and start the rendering. We can also create a color bar to aid us in visualization. To create a color bar we create a Scalar Bar Actor, add lookup table to it and feed the output to Scalar Bar Wiget.

{% highlight python %}
# The property describes how the data will look
ren = vtk.vtkRenderer()
renWin = vtk.vtkRenderWindow()
renWin.AddRenderer(ren)
iren = vtk.vtkRenderWindowInteractor()
iren.SetRenderWindow(renWin)

ren.AddVolume(volume)
ren.SetBackground(0, 0, 0)  #Set background color to black
renWin.SetSize(1600, 900)  #Set the display size 

# create the scalar_bar
scalar_bar = vtk.vtkScalarBarActor()
scalar_bar.SetOrientationToHorizontal()
scalar_bar.SetLookupTable(colorTransferFunction)
scalar_bar.SetMaximumWidthInPixels(100)
# create the scalar_bar_widget
scalar_bar_widget = vtk.vtkScalarBarWidget()
scalar_bar_widget.SetInteractor(iren)
scalar_bar_widget.SetScalarBarActor(scalar_bar)
scalar_bar_widget.On()


iren.Initialize() #Initialize and start rendering 
renWin.Render()
iren.Start()
{% endhighlight %}

That’s it. We are done. Run the code and you should see something like this. 
![3-D Output](/images/vtkImg/Dark_matter_density.png)
<iframe width="740" height="360" src="https://www.youtube.com/embed/LcYGXKOZiQU" frameborder="0"> </iframe>


## Final Thoughts  

This picture makes sense. Since dark matter isn't really interacting, the spherically symmetric shape isn't a surprise. Also, it might have something to do with the fact that it is a small simulation with only 1 galaxy cluster.

There is a higher density of dark matter at the center which I believe, is due to the fact there is much larger density of galaxies in the center of a cluster. We could have seen patterns in the density structure, but alas, the resolution is simply not good enough. We would probably need something like 1500x1500x1500 sized data to being seeing those finer details. 


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
