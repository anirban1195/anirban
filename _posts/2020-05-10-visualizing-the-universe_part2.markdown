---
layout: post
title:  "Visualizing the Universe with VTK(Part 2)"
date:   2020-08-11 20:00:00 -0600
categories: 10 Min Read
---
Last time we saw how volume render of dark matter. Using the exact same set of codes, we can do volume rendering of all the other quantities that are present in the data like Temperature or magnitude Magnetic Field. This time lets try and visualize vectors. The codes can be found [here](https://github.com/anirban1195/vtk/blob/master/create_Bfield_data.py) and [here](https://github.com/anirban1195/vtk/blob/master/bField_rendering.py)

## Vector Visualization 

The most common way of vector visualizations are streamlines,pathlines, timelines or streaklines. I will not go into the details of each one but you can easily look them up. Very briefly
* Streamlines : trajectory of massless particle in a steady flow or path of a particle in a frozen unsteady field 

* Pathlines: trajectory of massless particle in an unsteady flow 

* Timelines : connect particles released simultaneously at discrete time-steps 

* Streaklines: continuously inject particles at a fixed point in the flow, connect consecutive particles 

I will deal only with streamlines here for the obvious reason that the vector fields in the dataset we are dealing with can be considered largely frozen unsteady field. This is because each voxel is a few kiloparsec and the average field in that voxel is changing very slowly over human timescales.

## Data Pre-Processing 

Let’s start by trying to visualize the magnetic field. To use VTK for  vector fields, we need to create a new data set where each voxel has a vector associated with it. In our original data we have 3 magnetic field numbers associated with each voxel, i.e Bx, By, Bz. We need to explicitly tell VTK those are the 3 components of a vector. We start by importing libraries and recording the components of the magnetic field

{% highlight python %}
import h5py 
import numpy as np 
from numpy import mgrid, empty 
from tvtk.api import tvtk,write_data 
import sys 
with h5py.File('snap_s138.hdf', 'r') as f: 
    Bx = list(f.keys())[0]
    By = list(f.keys())[1]
    Bz = list(f.keys())[2]

    # Get the data
    datax = np.array(f[Bx])
    datay = np.array(f[By])
    dataz = np.array(f[Bz])

{% endhighlight %} 
Now we create a grid. This can be done by using mgrid in numpy. We need a 320x320x320 grid, same as our data size. 
{% highlight python %}
# Generate some points. 
x, y, z = mgrid[0:320,0:320, 0:320] 
{% endhighlight %} 

Now comes a slightly confusing part. To make a grid of say 100x100x100 points where each point has a given value of x,y and z, you will need a matrix of dimension (100,100,100,3). Think of it this way, the slice (100,100,100,0) holds all the x values of the (100,100,100) grid. The slice (100,100,100,1) holds all the y values and so on. This might take a minute to sink in, especially if you have not done this before. Similar arguments can be made for vectors. So, to generate the real points and vectors we do 
{% highlight python %}
# The actual points. 
pts = empty(z.shape + (3,), dtype=np.int16) 
pts[..., 0] = x 
pts[..., 1] = y 
pts[..., 2] = z 

# Some vectors 
vectors = empty(z.shape + (3,), dtype=np.float32) 
vectors[..., 0] = datax 
vectors[..., 1] = datay 
vectors[..., 2] = dataz 

{% endhighlight %} 

The ordering in vtk is z,x,y. So we need to flip the arrays. I am not very sure of this part but that is what I figured out from the documentation. If you results don’t match what you expect, you might need to do slight modification here 

{% highlight python %}
#Transposing because the order is z,y and x 
pts = pts.transpose(2, 1, 0, 3).copy() 
pts.shape = int(pts.size / 3), 3 #Flattening   

vectors = vectors.transpose(2, 1, 0, 3).copy() 
vectors.shape = int(vectors.size / 3), 3 #Flattening 

{% endhighlight %} 

We flatten the array to (N,3) because that’s what is required by tvtk. Now we can create a structured grid of dimension (320,320,320) and put in all the points and vectors. Each point has 6 numbers associated with it. The x,y and z values of the point and  the x , y and z components of the vectors at that point. 

{% highlight python %}

sg = tvtk.StructuredGrid(dimensions=x.shape, points=pts) 
sg.point_data.vectors = vectors 
sg.point_data.vectors.name = 'vector_field' 
write_data(sg, 'magField_vectors.vtk') 

{% endhighlight %} 


## Visualization 

Now we need to set the points from where our streamlines start. We can select any point randomly, but it is often recommended to have some sort of orientation of the points. That helps in visualization. I will select points uniformly spaced along the x,y and z axes at the center. This is because there is maximum density of matter at the center. Consequently, the B field is also strongest at the center. The code to this is 

{% highlight python %}
import h5py 
import sys
import vtk 
import numpy as np 
from vtk.util.misc import vtkGetDataRoot 
filename = 'magField_vectors.vtk'
reader = vtk.vtkStructuredGridReader()
reader.SetFileName(filename)
rakes = [ 
             vtk.vtkLineSource(), 
             vtk.vtkLineSource(), 
             vtk.vtkLineSource() 

             ] 

rakes[0].SetPoint1(160, 100, 160) 
rakes[0].SetPoint2(160, 200, 160) 
rakes[0].SetResolution(50)   

rakes[1].SetPoint1(100, 160, 160) 
rakes[1].SetPoint2(200, 160, 160) 
rakes[1].SetResolution(50)      

rakes[2].SetPoint1(160, 160, 100) 
rakes[2].SetPoint2(160, 160, 200) 
rakes[2].SetResolution(50) 

{% endhighlight %}
What we have done is used vtkLineSource to define our starting points. For example, rakes[0] is a set of 50 uniformly spaced points along the y axes from (160, 100, 160) to (160, 200, 160) . Now we can go ahead and create the streamlines 
{% highlight python %}
streamLineArr=[] 

for i in range(0, len(rakes)): 
	streamLine = vtk.vtkStreamTracer() 
	streamLine.SetInputConnection(reader.GetOutputPort()) 
	streamLine.SetSourceConnection(rakes[i].GetOutputPort()) 
	streamLine.SetMaximumPropagation(3000) 
	streamLine.SetIntegrationDirectionToForward() 
	streamLine.SetIntegratorTypeToRungeKutta4() 
	streamLineArr.append(streamLine) 

{% endhighlight %}
The lines should again be pretty self explanatory. For each set of points we define a vtkStreamTracer. We feed it the vector data we have and the starting points. Additionally we also tell it to integrate upto 3000 steps and use RK4 for integration.  

The hard part is now done. We can go ahead and create the rended window and the rendered and add these streamlines.  

{% highlight python %}
# Create the standard renderer, render window and interactor 
ren = vtk.vtkRenderer() 
renWin = vtk.vtkRenderWindow() 
renWin.AddRenderer(ren) 
iren = vtk.vtkRenderWindowInteractor() 
iren.SetRenderWindow(renWin) 
n=len(rakes) 
stream_actorArr=[] 

for j in range(n): 
	streamLineMapper = vtk.vtkPolyDataMapper() 
	streamLineMapper.SetInputConnection(streamLineArr[j].GetOutputPort()) 
	streamLineMapper.SetScalarModeToUsePointFieldData() 
	streamLineMapper.ScalarVisibilityOn() 
	#streamLineMapper.SelectColorArray(0) #Use this line if you want your streamlines to be colored  
	streamLineActor = vtk.vtkActor() 
	streamLineActor.SetMapper(streamLineMapper) 
	streamLineActor.VisibilityOn() 
	stream_actorArr.append(streamLineActor) 
	ren.AddActor(streamLineActor) 

iren.Initialize() 
renWin.Render() 
iren.Start() 

{% endhighlight %}

You should be able to see the streamlines now. They should look something like this 
<iframe width="740" height="360" src="https://www.youtube.com/embed/83uM7WChTTg" frameborder="0"> </iframe>


## Adding More Context 
These lines do not make much sense by themselves. Honsetly, they look jumbled mess of wires. For some context we can render some other quantity. We will render the total matter density. Depending on your needs you can select some other quantity as well. To do this we can simply use the codes from the previous post with slight modifications. We will need to add the following line before the rendering actually begins I.e before the line iren.Initialize

{% highlight python %}
#Total matter VR part  

filename = 'snap_s138.hdf'
with h5py.File(filename, 'r') as f: 

	# List all groups 
	idx = list(f.keys())[4] 
	# Get the data 
	data = list(f[idx]) 
#Scale the data 
data=np.array(data) +1 
data = np.log10(data)*1 
dataImporter = vtk.vtkImageImport() 
data_string = data.tostring() 
dataImporter.CopyImportVoidPointer(data_string, len(data_string)) 
print (len(data_string)) 

dataImporter.SetDataScalarTypeToFloat() 
dataImporter.SetNumberOfScalarComponents(1) 
dataImporter.SetDataExtent(0, 319, 0, 319, 0, 319) 
dataImporter.SetWholeExtent(0, 319, 0, 319, 0, 319) 

# Create transfer mapping scalar value to opacity 

opacityTransferFunction = vtk.vtkPiecewiseFunction() 
opacityTransferFunction.AddPoint(0, 0.0) 

opacityTransferFunction.AddPoint(0.15, 0.0) 
opacityTransferFunction.AddPoint(0.16, 0.1) 
opacityTransferFunction.AddPoint(0.25, 0.1) 
opacityTransferFunction.AddPoint(0.36 ,0.0) 

opacityTransferFunction.AddPoint(1, 0.0) 
opacityTransferFunction.AddPoint(1.01, 0.2) 
opacityTransferFunction.AddPoint(1.15, 0.2) 
opacityTransferFunction.AddPoint(1.16 ,0.0) 

opacityTransferFunction.AddPoint(1, 0.0) 
opacityTransferFunction.AddPoint(1.71, 0.2) 
opacityTransferFunction.AddPoint(1.85, 0.2) 
opacityTransferFunction.AddPoint(1.86 ,0.0) 

opacityTransferFunction.AddPoint(2, 0.0) 
opacityTransferFunction.AddPoint(2.01, 0.2) 
opacityTransferFunction.AddPoint(2.15, 0.2) 
opacityTransferFunction.AddPoint(2.16 ,0.0) 

opacityTransferFunction.AddPoint(2, 0.0) 
opacityTransferFunction.AddPoint(2.71, 0.2) 
opacityTransferFunction.AddPoint(2.85, 0.2) 
opacityTransferFunction.AddPoint(2.86 ,0.0) 

opacityTransferFunction.AddPoint(3, 0.0) 
opacityTransferFunction.AddPoint(3.01, 0.2) 
opacityTransferFunction.AddPoint(3.15, 0.2) 
opacityTransferFunction.AddPoint(3.16 ,0.0) 

opacityTransferFunction.AddPoint(4, 0.0) 
opacityTransferFunction.AddPoint(4.01, 0.2) 
opacityTransferFunction.AddPoint(4.15, 0.2) 
opacityTransferFunction.AddPoint(4.16 ,0.0) 

colorTransferFunction = vtk.vtkColorTransferFunction() 
colorTransferFunction.AddRGBPoint(0,0.2,0.5,0.8) 
colorTransferFunction.AddRGBPoint(2,1,1,0) 
colorTransferFunction.AddRGBPoint(4,1,0,0) 

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


# The volume holds the mapper and the property and 
# can be used to position/orient the volume 
volume = vtk.vtkVolume() 
volume.SetMapper(volumeMapper) 
volume.SetProperty(volumeProperty) 
volume = vtk.vtkVolume() 
volume.SetMapper(volumeMapper) 
volume.SetProperty(volumeProperty) 

ren.AddVolume(volume)
ren.SetBackground(0, 0, 0)
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

{% endhighlight %}

Finally, you should end up with something like this.
<iframe width="740" height="360" src="https://www.youtube.com/embed/oKeAV4iZ--Y" frameborder="0"> </iframe>
Now you can change the points at which your streams are seeded and hopefully get interesting insights.
This is a good place for me stop since I don’t know much about how magnetic fields in astronomical context. However, take the result of the streamlines with a grain of salt since each voxel is about the size of half the milky way.

There is a bunch of other stuff you can do. Just out of curiosity I made a volume rendering of the components of velocity towards the cluster centre and they look like this.
<iframe width="740" height="360" src="https://www.youtube.com/embed/8qV3Z23w0dg" frameborder="0"> </iframe>
Red represents infalling and blue is matter being ejected. Clearly most of the matter is infalling.

## Notes 
The file sizes are fairly large, in the GB range. While running this code, I found I needed about 6 GB or so of free memory. In case you have a smaller RAM, you might want to use the low resolution version of the files [here](https://b2share.eudat.eu/records/a9ca0edb44c24b459674c6b7d552f887)
