# Neural Radiance Fields (NeRF) on custom synthetic datasets


## Plan of Action

0. Prerequisites
      - Ray Tracing
      - Ray Casting
      - Ray Marching

1. Understanding NeRF
     - Volumetric Scene Representation
     - Volume Rendering
     - Improvement 1: Positional Encoding
     - Improvement 2: Hierarchical Volume Sampling

2. 3D Model using Blender

3. Training NeRF


---------------
## 0. Prerequisites





--------------------------

## 1. Understanding NeRF






### 1.1 Volumetric Scene Representation
What has been done before NeRF is to have a set of images and use 3D CNN to predict a discrete volumetric representation such as a **Voxel Grid**. Though this technique has demonstrated impressive results, computing and storing these large voxel grids can  become computationally expensive for large and high-resolution scenes. What NeRF does is represent a scene as a **continuous** ```5D function``` which consists of **spatial 3D location** ```x = (x,y,z)``` of a point and the **2D viewing direction** ```d = (θ, φ)```. This is the **input**.

<p align="center">
  <img src="https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/29a33610-e1d3-4800-a1eb-2e2f01777cf0" width="70%" />
</p>


By using the 5D coordinates along camera rays as input, they can then represent any arbitrary scene as a **Fully Connected neural network (MLP)** - ```9 layers and 256 channels each```. By feeding those locations into the MLP, they produce the **emitted color** in ```c = (r,g,b)```  and the **volume density**, ```σ```. This is the **output**.

<p align="center">
  <img src="https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/895e4a50-006c-452f-a3c1-f1fbaf610cd2" />
</p>

From the function above, we want to optimize the **weights** ![CodeCogsEqn (4)](https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/c0bcb685-e5b1-4ffb-af58-5060316a7453) that effectively associates each 5D coordinate input with its respective volume density and emitted directional color that represents the radiance.


<p align="center">
  <img src="https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/673bcdde-e529-421c-ab15-fe0f8ff66475" width="70%" />
</p>
<div align="center">
    <p>Image Source: <a href="https://en.wikipedia.org/wiki/Spherical_coordinate_system">Spherical coordinate system</a></p>
</div>

Let's explain the overall pipeline of the NeRF architecture:

**a)** Images are generated by selecting specific 5D coordinates that include both spatial location and the direction in which the camera is looking, all along the paths of camera rays.

**b)** Using these positions, we input them into a Multi-Layer Perceptron (MLP) to generate both color and volume density information.

**c)** We apply volume rendering techniques to combine these values into a final image.

**d)** Since this rendering function is differentiable, we can improve our scene representation by minimizing the difference between the images we synthesize and the ground truth images we've observed

Moreover, the author argues that they promote multiview consistency in the representation by constraining the network to estimate the **volume density**, σ as a function of the s**patial position** (**x**) exclusively. At the same time, they enable the prediction of RGB color (**c**) as a function of both the **spatial position** (**x**) **and** the **viewing direction (d)**.


**Note:**

1. θ (theta) and φ (phi) represent the angular coordinates of a ray direction in **spherical coordinates** as shown below.
2. The output density represents the probability distribution of how much of a 3D point in the scene is occupied by the object or scene surface. More precisely, it indicates whether a particular 3D point along a viewing ray intersects with the object's surface or not.

<p align="center">
  <img src="https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/06c54b78-3cc2-40fd-83ec-18527e3903f4" width="30%" />
</p>
<div align="center">
    <p>Image Source: <a href="https://en.wikipedia.org/wiki/Spherical_coordinate_system">Spherical coordinate system</a></p>
</div>







### 1.2 Volume Rendering
Recall that density, σ, can be binary, where it equals ```1``` if the point is on the object's surface, i.e., it intersects with the scene geometry, and ```0``` if it is in empty space. Hence, everywhere in space, there is a value that represents density and color at that point in space.

<p align="center">
  <img src="https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/5e70e0bc-ac28-4816-b27f-f53b0d97b501" width="60%" />
</p>
<div align="center">
    <p>Image Source: <a href="https://en.wikipedia.org/wiki/Spherical_coordinate_system">Spherical coordinate system</a></p>
</div>


We start by shooting a ray (camera ray) in our scene as shown below by Ray 1 and Ray 2. The equation of the camera ray is dependent on the origin, **o**, and the viewing direction, **d** for different time t.

<p align="center">
  <img src="https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/5aa4118c-f427-4a19-a3c2-de52d4a3c30e" width="10%" />
</p>

We then sample a few points along the ray. For each point, we record the density and color at this point in space. We calculate the expected color as such:

<p align="center">
  <img src="https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/d7aa1eb3-722f-4ec2-83f9-26e18cde34b8" width="30%" />
</p>

- From the equation above, we observe we have the product of the density at point **r**(t): ```(σ(r(t)))``` which is independent of viewing direction **d** and the color at point **r**(t) from viewing direction **d**: ```(c(r(t),d))```. This means that if the density is 0, the color has no impact. But if we have a high density, the color has a bigger weight.

- We also have the term ```T(t)``` which is defined as the ```accumulated transmittance```. This refers to how much light is transmitted or attenuated along a viewing ray as it passes through the scene. So basically we will compute the density accumulated. Consider a scenario where there are two objects, A and B, positioned such that A is situated behind B. In this arrangement, A becomes occluded by B. Consequently, as a ray traverses through B, density accumulation occurs along that ray. When the ray subsequently intersects with A, it won't significantly affect the color because the density has already been accumulated. However, if the ray extends into empty space and encounters another object, it will have an impact on the final color because, in this case, density accumulation has not yet taken place, and the first object encountered will influence the color as the ray progresses. In other words, it quantifies the probability that the ray travels from ```tn``` to ```t``` without encountering any other particles along its path.

<p align="center">
  <img src="https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/d408153b-0ec9-443f-996b-778f56b6aa6a" width="20%" />
</p>

1. To compute the color of a camera array that passes through the volume we need to estimate a continuous 1d line integral along that ray.

2. They do this by querying the MLP at multiple sample points within the range of starting and ending distances, denoted as t1 and tn.

3. The author does not solve this integration numerically but instead uses the ```quadrature rule```.

<p align="center">
  <img src="https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/e74f3013-bae2-487d-8485-f8d73f7643a9" width="30%" />
</p>

4. This estimation process computes the color ![CodeCogsEqn (12)](https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/38bc9dd1-b888-4520-b263-e9f2f5158c64) of any camera ray by summing up contributions from each segment of the ray's path.

<p align="center">
  <img src="https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/6eec67d0-3eb1-4b24-bbd5-73b575671d6e" width="25%" />
</p>

5. Each contribution includes the color of the segment ![CodeCogsEqn (13)](https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/8e1d0b01-a684-45d9-877c-50ddae651da2), which is weighted by the accumulated transmittance, ![CodeCogsEqn (14)](https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/2f47303e-24b2-47f4-a512-3f96851a521c), which computes how much light is blocked earlier along the ray _and_ ![CodeCogsEqn (15)](https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/8262a74f-dfc8-43e4-918b-585bdcf9d1a3) which is how much light is contributed by ray segment i, which is a function of the segment's length and its estimated volume density.

<p align="center">
  <img src="https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/4547ce16-bb95-49cc-a316-b5d1865d368a" width="17%" />
</p>

<p align="center">
  <img src="https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/52876cd2-5491-4278-b0be-9ade4c42d677" width="17%" />
</p>

The author also argues that they allow the color of any 3D point to vary as a function of the **viewing direction** as well as **3D position**. If we change the direction inputs for a fixed (x,y,z) location, we can visualize what view-dependent effects have been encoded by the network. Below is a visualization of two different points in a synthetic scene. It demonstrates how for a fixed 3D location adding view directions as an extra input allows the network to represent realistic view-dependent appearance effects


<p align="center">
  <img src="https://github.com/yudhisteer/Training-a-Neural-Radiance-Fields-NeRF-/assets/59663734/2cbb0834-8d8e-4ef1-a7e1-0c0eff2d5092" width="60%" />
</p>






### 1.3 Improvement 1: Positional Encoding


### 1.4 Improvement 2: Hierarchical Volume Sampling

----------

## 2. 3D Model using Blender






---------------

## 3. Training NeRF



--------------------------






## References
1. https://www.youtube.com/watch?v=CRlN-cYFxTk&ab_channel=YannicKilcher
2. https://www.youtube.com/watch?v=JuH79E8rdKc&ab_channel=MatthewTancik
3. https://www.youtube.com/watch?v=LRAqeM8EjOo&ab_channel=BENMILDENHALL
4. https://www.fxguide.com/fxfeatured/the-art-of-nerfs-part1/?lid=7n16dhn58fxs
5. https://www.youtube.com/watch?v=CRlN-cYFxTk&t=1745s&ab_channel=YannicKilcher
6. https://www.fxguide.com/fxfeatured/the-art-of-nerfs-part-2-guassian-splats-rt-nerfs-stitching-and-adding-stable-diffusion-into-nerfs/
7. https://www.youtube.com/watch?v=nCpGStnayHk&ab_channel=TwoMinutePapers
8. https://www.youtube.com/watch?v=4NshnkzOdI0&list=PLlrATfBNZ98edc5GshdBtREv5asFW3yXl&index=2&ab_channel=TheCherno
9. https://www.youtube.com/watch?v=AjkiBRNVeV8&ab_channel=TwoMinutePapers
10. https://www.youtube.com/watch?v=NRmkr50mkEE&ab_channel=TwoMinutePapers
11. https://www.youtube.com/watch?v=ll4_79zKapU&ab_channel=BobLaramee
12. https://www.youtube.com/watch?v=g50RiDnfIfY&t=112s&ab_channel=PyTorch
13. https://www.peterstefek.me/nerf.html#:~:text=NeRF%20relies%20on%20a%20very,is%20useful%20for%20understanding%20NeRF.
14. https://datagen.tech/guides/synthetic-data/neural-radiance-field-nerf/#
15. https://dtransposed.github.io/blog/2022/08/06/NeRF/
