---
layout: article
title: Streamline Visualization of 3D Wind
description: An application for visualizing 3D wind data as streamlines.
cover: /assets/images/3dwind/cover.png

article_header:
  background_image:
    src: /assets/images/3dwind/cover.png
---
<!--more-->

## Overview
This unnamed application was a prototype for creating multiscale visualizations of 3D vector Earth data using streamlines.
I created the visualization tool as a course project for _CPSC 615: Computational Techniques for Graphics and Visualization_, which I took in the second year of my Master's degree.
As a prototype, I chose to focus on visualizing wind data for purposes of the project.

The application was developed in C++ and OpenGL; Eigen3 and Dear ImGui were used for linear algebra and GUI widgets, respectively.
The source code for the project can be viewed [here](https://github.com/benjaminulmer/wind-streamlines).

### Video Demo
<div>{%- include extensions/youtube.html id='Ym-1vtnu8Ug' -%}</div>

### Data Source
The wind data used in this project was obtained from the [ERA5](https://www.ecmwf.int/en/forecasts/datasets/reanalysis-datasets/era5) dataset.
The data comes sampled on a 0.25-degree latitude-longitude grid with 37 pressure (altitude) levels.
Pressure levels are represented as a pressure in mbars, which can be converted to an equivalent altitude in meters.
However, the pressure levels are not uniformly spaced altitudes, with levels near the Earth's surface closer together than those higher in the atmosphere.
Wind velocity components are represented as m/s East, m/s North, and Pa/s (vertical component).
I used trilinear interpolation to sample the dataset.

## Streamlines
[Field lines](https://en.wikipedia.org/wiki/Field_line) are the family of curves that are tangent at every point to the vector field they represent.
Streamlines are simply field lines for flow data and have long been studied as a method for representing and visualizing flow data.
Compared to simple glyph-based visualizations, streamlines better represent the continuity of flow and are less prone to visual clutter---though this is still a challenge, especially in 3D.
For steady flow, streamlines are equivalent to [pathlines and steaklines](https://en.wikipedia.org/wiki/Streamlines,_streaklines,_and_pathlines); however, for unsteady flow, streamlines only represent the data at a single snapshot in time.

### Integration
Calculating field lines requires computing approximate-solutions to the initial value problem

$$\frac{dy}{dx} = f(t,y), \quad y(t_0) = y_0,$$

where $y$ is an unknown function that we wish to evaluate at successive values of $t$.
The initial values $t_0$ and $y_0$ are given, along with the function $f$.
The [Runge-Kutta](https://en.wikipedia.org/wiki/Runge%E2%80%93Kutta_methods) family of methods is commonly used to find solutions to ordinary differential equations such as this one.
Determining an appropriate step size for these methods is important for getting sufficiently accurate results without needlessly increasing the computational cost.
However, the amount of error introduced by these methods depends not only on the step size but also on the vector field's characteristics, which makes choosing an appropriate fixed step size difficult.
One solution to this is to use an adaptive step size, where the error of a given step is estimated, and the step size is changed accordingly.
Estimating error requires computing the next step using two different methods with similar error constants, which approximately doubles computation time.
Embedded methods address this by using shared computations between the two calculations, which can allow for an adaptive step size to be determined with only one extra computation compared to a fixed step size method.
For this project, I used the [Runge-Kutta-Fehlberg method (RKF45)](https://en.wikipedia.org/wiki/Runge%E2%80%93Kutta%E2%80%93Fehlberg_method), which is an order four embedded method.

### Error Tolerance
While implementing an RKF45 integrator is simple, determining an acceptable error tolerance is non-trivial.
A lower error tolerance creates more accurate and smooth streamlines; however, this also increases integration time and the computed streamlines' memory footprint.
This tradeoff is particularly pertinent for global Earth data due to the scale of the data compared to the scale of the domain.
For wind data, in particular, the largest values are on the order of a few hundred m/s compared to the thousands of kilometres of integration needed for streamlines.
Thus, to integrate these streamlines in a reasonable amount of time, large time steps need to be taken.
Through trial and error, I found an error tolerance of 1 km produced acceptable results in most cases, although sometimes the resulting streamlines were less smooth than desired.
This issue is especially noticeable in regions of high vorticity.

![Jagged streamlines in region of high vorticity](/assets/images/3dwind/not-smooth.png)
{: style="display: block; margin-left: auto; margin-right: auto; width: 50%;"}

These artifacts could be detected and corrected for; however, doing so ended up being outside the scope of the project.

In addition to the error tolerance, I also specified a maximum step size of 10,000 seconds for each iteration.
Without such a limit, the step size would occasionally become excessively large in regions of near-constant velocity and not recover.
Again, I chose this value through a process of trial and error.

### Position Updating
Runge-Kutta integrators require updating a given position by a vector.
Positions on the Earth can be represented in spherical or Cartesian coordinates, but recall that for the ERA5 dataset, wind velocity is stored in m/s East, m/s North, and Pa/s.
Thus, we must perform coordinate and unit conversions to update a position on Earth by such a velocity vector.

Regardless of if points are stored in spherical or Cartesian coordinates, the first step is to convert the East and North components of velocity---which are tangential velocities---to angular velocities.
Let $v$ and $\omega$ refer to tangential and angular velocities, respectively.
We will use subscript $e$ for the East component and $n$ for the North.

For $v_n$, the axis of rotation is located at the centre of the Earth, since lines of longitude are great circles.
Therefore,

$$\omega_n = \frac{v_n}{R + a},$$

where $R$ is the radius of the Earth and $a$ is the altitude of the point, both in meters.
Calculating $v_e$ is slightly more involved since lines of latitude are small circles: we must first calculate the radius ($r$) of the small circle.
If points are stored in spherical coordinates, this is simply $r = (R + a) \cos\phi$, where $\phi$ is the latitude of the point.
For Cartesian coordinates, $r = \sqrt{x^2 + z^2}$, assuming the y-axis is aligned with the poles.
In either case, we get that 

$$\omega_e = \frac{v_e}{r}.$$

With angular velocities, spherical positions can be updated directly, and Cartesian positions can be updated by rotating about the respective axis.
For the vertical component, Pa/s are converted to m/s using the International Standard Atmosphere [formula for pressure altitude](https://en.wikipedia.org/wiki/Pressure_altitude):

$$145366.45 \left( 1 - \left( \frac{\mathrm{pressure~in~mbars}}{1013.25} \right)^{0.190284}  \right).$$

The result of this formula is in feet, which is converted to meters by multiplying by 0.3048.
Pa are converted to mbars by multiplying by 0.01.

## Streamline Seeding
With the ability to integrate individual streamlines, a strategy is needed to create a set of streamlines that result in a good representation of the entire flow field.
There are four conflicting goals of most streamline visualizations:

1. **Coverage:** interesting regions and critical points in the flow should not be missed
2. **Uniformity:** spacing of streamlines should be mostly uniform to aid in visual interpolation
3. **Continuity:** streamlines should be as long as possible to show continuity in the flow
4. **Non-cluttered:** too many streamlines result in a cluttered visualization. This is especially problematic in 3D where occlusion becomes an issue

Creating a seeding algorithm that results in a good balance between the above properties is a non-trivial problem, with much research done in this area.
For this project, I chose to use the algorithm proposed by [Jobard and Lefer (1997)](https://www.researchgate.net/profile/Bruno_Jobard/publication/2825680_Creating_Evenly-Spaced_Streamlines_of_Arbitrary_Density/links/55bb6f4b08aed621de0d9486/Creating-Evenly-Spaced-Streamlines-of-Arbitrary-Density.pdf).
The main motivations for using this algorithm were its simplicity and the ease at which it can be extended to multiresolution ([Jobard and Lefer (2001)](https://www.researchgate.net/profile/Bruno_Jobard/publication/221546343_Multiresolution_Flow_Visualization/links/0046351dfb5bcb61db000000.pdf)).
A brief overview of the algorithm is as follows:

1. Let _L_ be a queue of streamlines
2. Integrate a streamline and add it to _L_
3. Pop the first element of _L_: call this _S_
4. For each point on _S_, generate candidate seeds that are some separation distance away
5. Integrate each candidate seed until it gets within some distance of another streamline
6. If the streamline is longer than some minimum length, keep it and add it to _L_. Otherwise, discard the streamline and continue
7. Repeat steps 3--6 until _L_ is empty

For global-scale streamlines, I found a separation distance of 200 km and a minimum length of 1000 km produced nice results.
I also limited the integration of streamlines to a maximum distance of 10,000 km, both forward and backward, to prevent the algorithm from bottlenecking on any single streamline.

### Custom Distance Metric
Applying this algorithm directly to our dataset, we immediately notice an issue: streamlines only appear at a certain altitude for any given region.
This issue is a consequence of the small vertical domain of the data---about 32 km.
Therefore, for streamlines to cross above or below another one, the seeding algorithm's separation distance needs to be less than 32 km.
At a global scale, such a small separation distance results in far too many streamlines.
Since we want all scales to highlight the 3D characteristics of the flow, a different solution is needed.

To address this issue, I employed a custom distance metric that weights vertical distance more than the geodesic distance along the sphere.
Given two points, $p_1$ and $p_2$, the distance between them is defined as follows.
Let $a$ be the minimum altitude of $p_1$ and $p_2$.
Then, the geodesic distance between $p_1$ and $p_2$ is

$$g = (R + a) \cdot \theta,$$

where $\theta$ is the angle between the two points.
The vertical distance, $v$, between the points is simply the difference in their altitude.
The final distance is given by 

$$\sqrt{g^2 + (\beta v)^2},$$

where $\beta$ is a constant weighting factor.
I found that values for $\beta$ in the range 25--40 produced good results.
It is worth noting that this custom distance metric is used both when comparing the distance between points and when generating new seed points from existing streamlines.
The following images show the results of the seeding algorithm without (left) and with (right) this custom distance metric:

![Streamlines all at one altitude](/assets/images/3dwind/one-alt.png)
{: style="float: left; margin-left: auto; margin-right: auto; width: 49.9%; padding-right: 1%"}
![Many steamlines crossing over at different altitudes](/assets/images/3dwind/many-alts.png)
{: style="float: left; margin-left: auto; margin-right: auto; width: 49.9%; padding-left: 1%"}

While the set of streamlines obtained using the custom distance metric appears overly cluttered and messy, this issue will be addressed with changing how the streamlines are [rendered](#rendering), not seeded.

### Multiresolution Seeding
Thus far, the seeding strategy produces good results when viewed at a global scale; however, when zooming in to a more local scale, the set of streamlines becomes too sparse.
To create a multiscale visualization, we should integrate/display more streamlines as the user zooms in on the Earth.

As mentioned previously, one of the reasons this seeding algorithm was chosen is how easily it can be extended to produce multiple resolutions of streamlines.
The multiresolution extension works by adding additional streamlines to the output of the base algorithm.
Using the computed set of streamlines as the queue _L_, steps 3--7 are simply repeated with a smaller separation distance and minimum length.
For this dataset, I found scaling the separation distance and minimum length by a factor of 0.8 for each successive resolution produced good results.
In the application, several resolutions of streamlines are calculated as a pre-processing step; then, the user can control the number of resolutions displayed during runtime.

### Acceleration
Recall that the seeding algorithm requires finding the minimum distance between the streamline being integrated and every other streamline computed thus far.
The brute force solution of comparing against every point on every other streamline is far too inefficient, especially since this must be done for each newly calculated point during integration.
Thus, a spatial partitioning data structure is needed to allow for efficient distance queries by avoiding unnecessary comparisons.
I used a simple uniform voxel grid with a hash lookup that maps the index of a voxel to a list of points contained within.
In code, this can be accomplished with standard library containers:
```c++
std::unordered_map<size_t, std::vector<Eigen::Vector3d>>
```
With the voxels' dimensions set to the seeding algorithm's separation distance, each newly calculated point only needs to be compared against those in the 3x3x3 cube of voxels centred on the voxel that contains said point.
While such a voxel grid is likely not the ideal data structure for the task---especially considering the custom distance metric used---I chose it for its simplicity and ease of implementation.

## Rendering
Regardless of how streamlines are seeded, the rendering of said streamlines dramatically impacts the final visualization.
This is especially true for 3D vector fields, where depth perception, occlusion, and clutter become significant issues that need addressing.
In this project, I implemented several rendering options for the streamlines to aid users' perception of key aspects of the data.

### Geometry and Illumination
Streamlines are typically modelled as either simple polylines or full 3D geometry (typically a cylinder).
3D geometry has well-defined surface normals used in typical illumination models; however, this comes at an overhead cost for calculating and storing the geometry.
3D geometry also has an absolute thickness (radius of the cylinder), whereas for a multiscale visualization, having streamlines with constant thickness in _screen space_ is much more desirable.
Alternatively, polylines are a simple and memory minimal representation of streamlines that can easily be rendered with a constant screen space thickness.
One drawback with this approach is that lines do not have a single surface normal for illumination calculations, but rather a plane of possible normals orthogonal to the tangent direction.
Despite this, it is possible to choose a single normal in a consistent manner that allows illumination models requiring a normal to be applied to the lines. 
For these reasons---and simplicity---I used polylines for representing streamlines in this project.

To illuminate the streamlines, I used the method proposed by [Zöckler et al. (1996)](https://www.researchgate.net/profile/Hans-Christian_Hege/publication/224208651_Interactive_Visualization_of_3D-Vector_Fields_Using_Illuminated_Streamlines/links/0c960515c53d4e8c8e000000.pdf), where the normal direction that is coplanar to the light direction is used as the surface normal.
With this definition, the values needed for Phong illumination can be calculated from the light, view, and _tangent_ vectors.
Let $L$ be the light, $V$ the view, $T$ the tangent, $N$ the normal, and $R$ the reflection direction (all pointing away from the surface).
From basic trigonometry, the expressions $N \cdot L$ and $V \cdot R$ can be expressed in terms of $L$, $T$, and $V$:

$$ N \cdot L = \sqrt{1 - \left( L \cdot T \right)^2 } $$

$$ V \cdot R = N \cdot L \sqrt{1 - \left( V \cdot T \right)^2 } - (L \cdot T) (V \cdot T) $$

Refer to the original paper for the derivation of these expressions; however, note that they are derived with $L$ pointed _toward_ the surface, which results in a slightly different final result for $V \cdot R$.
The figures below show streamlines with constant illumination (left), diffuse Lambertian reflection (right), and both diffuse reflection and Phong specular highlights (bottom).

![Streamlines with constant illumination](/assets/images/3dwind/no-illumination.png)
{: style="float: left; margin-left: auto; margin-right: auto; width: 49.9%; padding-right: 1%;"}
![Streamlines with diffuse Lambertian reflection](/assets/images/3dwind/diff.png)
{: style="float: left; margin-left: auto; margin-right: auto; width: 49.9%; padding-left: 1%"}
![Streamlines with diffuse reflection and Phong specular highlights](/assets/images/3dwind/diff-spec.png)
{: style="display: block; margin-left: auto; margin-right: auto; width: 50%;"}

### Altitude
Discerning the depth and locations of streamlines in 3D space is a significant challenge in these types of visualization.
In our case, the challenge is amplified by the fact the vertical domain of the dataset is only about 32 km, which means at a global scale, differences in altitude are imperceptible.
To address this, I implemented two simple effects to highlight the altitude of streamlines, which can be used either independently or together.
The first method applies a colour scale to the altitude of streamlines; the second scales the altitude of streamlines by some factor.
Additionally, streamlines can also be given a black outline---sometimes called a halo.
While this does not directly encode altitude information, it can help with depth perception nonetheless.
These effects are demonstrated in the images below: just colour (top left); just scaling (top right); both colour and scaling (bottom left); and streamline halos (bottom right).

![Streamlines with colour representing altitude](/assets/images/3dwind/color.png)
{: style="float: left; margin-left: auto; margin-right: auto; width: 49.9%; padding-right: 1%"}
![Streamlines with scaled altitude](/assets/images/3dwind/scale.png)
{: style="float: left; margin-left: auto; margin-right: auto; width: 49.9%; padding-left: 1%"}
![Streamlines with colour representing altitude and scaling](/assets/images/3dwind/both.png)
{: style="float: left; margin-left: auto; margin-right: auto; width: 49.9%; padding-right: 1%"}
![Streamlines with a black halo effect](/assets/images/3dwind/halos.png)
{: style="float: left; margin-left: auto; margin-right: auto; width: 49.9%; padding-left: 1%"}

The colours used for streamlines, the vertical scale factor, and the halo width (including no halo) can all be changed dynamically by the user during execution.

### Streamline Tails
While streamlines are tangent at all points to the flow field, solid lines convey no information about the speed of flow or which direction along the line it goes.
By adding transparency to sections of streamlines, we can address this issue by creating a tail effect that represents the direction and speed of flow along streamlines.
The main idea is to simulate particles that travel along the streamline, emitting a haze or smoke as they do.
The haze fades exponentially as time passes, creating a tail that follows the imaginary particles, indicating their direction and speed.
The effect also reduces clutter issues, as only a certain percentage of each streamline is visible.

To calculate the transparency for this effect, we must store the integration time of each point along the streamlines.
In other words, each point calculated during streamline integration also stores the total accumulated time step in addition to the computed position.
Let $t$ refer to this integration time.
Then, the transparency of fragments calculated in the shader is given by

$$\alpha^{-t \mod i},$$

where $i$ is the time interval that spaces particles on the same streamline and $\alpha$ is the percentage of the haze that remains after each time unit.
This effect creates a series of static particles and trails; however, it can be animated by augmenting the above definition with an accumulator $T$ for the total time that has progressed since starting the application.
I also added a scale factor $m$ that can be applied to $T$ to affect the animation's speed.
Such a scale factor is particularly useful for this dataset, as the animation is nearly static at actual speed due to the considerable integration time covered by a single streamline.
The updated formula for fragment transparency is

$$\alpha^{\left( T - t \mod i \right) / m}.$$

Like most other rendering parameters, the values of $q$, $m$, and $i$ can all be changed by the user at runtime.
The starting values are $q = 0.3$, $m = 30\,000$, and $i = 100\,000$, and result in the animations shown in the following two videos.

<div>{%- include extensions/video.html path='/assets/videos/3dwind/global.mp4' width='75%' -%}</div>
<br>
<div>{%- include extensions/video.html path='/assets/videos/3dwind/local.mp4' width='75%' -%}</div>

With adding transparency to the rendering equation, proper depth ordering of render calls is needed to avoid blending artifacts.
Since most streamlines have relatively little variation in altitude, I simply sort them by their average altitude.
While this does not eliminate all artifacts, I found it eliminated the vast majority of them and was more than acceptable, especially considering the algorithm's simplicity.

## Limitations
In its current state, the application still has several limitations.
Some of the limitations are easily addressed; however, others are more fundamental and would require reworking key functionalities.

While nearly all rendering effects for streamlines are calculated in the shaders and controlled by uniforms, colours are stored per-vertex and require updating the vertex buffers every time the user changes the colour settings.
Calculating the colour of streamlines could be done in the shader, as it is a function only of their altitude, but would require reimplementing the colour scale functionality in GLSL.
Due to time constraints, I chose to forgo this feature in favour of working on other aspects of the project.
Similarly, as mentioned previously, the speed of streamline integration could likely be improved with a better space partitioning data structure.
Doing so would be an exercise of implementing and benchmarking different data structures to see which one(s) work best for the data. 

More fundamental limitations relate to the streamline seeding algorithm used.
While the multiresolution seeding is a pre-processing step, it is still relatively slow and can take several minutes to finish.
The resolution of streamlines also does not update automatically based on the view, although logic for doing so based on camera parameters could be added.
Finally, the seeding algorithm does not necessarily capture interesting regions of the flow.
I did implement a method for detecting critical points in the flow data but ran out of time before I could incorporate this into the seeding algorithm.

## Reflection
This application is one of the more recent C++ projects I worked on; it is also one of the largest projects I did entirely independently.
Having never worked with vector field visualizations before, I learned a lot about sampling, Runge-Kutta methods, numerical stability, line rendering and visualization techniques, and different streamline seeding algorithms/approaches.
Also, despite being a course project under tight time constraints, I tried to put effort into making the codebase well designed and readable.
Looking back now, I see the code falls short in many aspects---especially with regards to some best practices for modern C++ that I was only vaguely familiar with at the time---although most of the worst offenders could be fixed considerably easily.

Concerning the resulting visualization, I am happy with how it turned out aesthetically, but I doubt its effectiveness for actually communicating the underlying data.
At a global scale, the streamlines give a decent impression of the data's overall patterns; however, the smaller scale views are much less useful.
Additionally, going into the project, I was hoping there would be more vertical variation in the resulting streamlines.
While there is some, the vertical component of velocity tends to be much less than the surface components; this results in any given streamline having close to constant altitude.
Still, some interesting streamlines did appear in certain datasets.
Finally, while I initially planned to extend this application to other Earth vector fields, such as ocean currents, finding such datasets proved to be a significant challenge.
