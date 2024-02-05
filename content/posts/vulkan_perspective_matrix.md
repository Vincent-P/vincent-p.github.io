+++
title = "The perspective projection matrix in Vulkan"
publishDate = 2021-01-02T00:00:00+01:00
aliases = ["/notes/20201216234910-the_projection_matrix_in_vulkan"]
lastmod = 2021-11-22T00:07:51+01:00
tags = ["vulkan", "graphics", "maths"]
draft = false
comments_url = "https://www.reddit.com/r/GraphicsProgramming/comments/kpkoht/the_projection_matrix_in_vulkan/"
maths = true
+++

The perspective projection matrix is crucial in computer graphics to display 3d points on a screen.
In most of the computer graphics/opengl/vulkan tutorials online there is only a brief mention of the `glm::perspective` function and its parameters, and quick "hacks" to make it work on Vulkan (Hello negative viewport and correction matrix).
In these notes I will try to explain the maths behind the perspective projection and give a matrix that works with Vulkan.
Don't worry you can still follow if you don't use Vulkan and adapt the formulas with **your** settings.

{{< figure rel="persp_matrix_frustum1.svg" >}}

![Figure 1: The frustum and the clip volume](/blog/persp_matrix_frustum1.svg)

The projection matrix is a transformation of the camera (or eye) space into clip space.
The clip space is a homogeneous space that is used to remove (or clip) primitives outside the viewport.
After clipping, the hardware will perform a "perspective division" that will transform the clip coordinates into normalized device coordinates by dividing each component by the 4th component w.


## Classic perspective with a near and far plane {#classic-perspective-with-a-near-and-far-plane}


### Projecting onto the near plane {#projecting-onto-the-near-plane}

The first step consists of projecting points in eye space on the near plane of the frustum.
Here is the eye space is a right-handed coordinate system and the camera is facing towards the -Z axis.
Let's start by finding the values of \\(x\\) and \\(y\\), the depth will be derived later.

![Figure 2: The frustum volume, the point (xe, ye, ze) gets projected in the near plane at coordinates (xp, yp, zp). The blue line is the Z axis.](/blog/persp_matrix_frustum2.svg)

The frustum is a pyramid with the camera as its apex: it forms similar triangles on the XZ and YZ planes.

Using the Intercept theorem (also known as Thales's theorem) it's easy to find the value of \\(x\_p\\) and \\(y\_p\\).


![Figure 3: Side view (YZ plane) of the frustum, the top view (XZ plane) is similar](/blog/persp_matrix_frustum3.svg)

The intercept theorem tells us that the ratio between a point projected on the near plane and the coordinate in eye space is the same: \\[ \frac{x\_p}{x\_e} = \frac{y\_p}{y\_e} = \frac{z\_p}{z\_e} = \frac{-n}{z\_e} \\]

\\(z\_p\\) will always be \\(-n\\) because we are projecting points on the near plane.

In the end we have:

\begin{align}
& x\_p = \frac{-n x\_e}{z\_e} = \frac{1}{-z\_e} n x\_e\\\\
& y\_p = \frac{-n y\_e}{z\_e} = \frac{1}{-z\_e} n y\_e\\\\
& z\_p = -n = \frac{1}{-z\_e} n z\_e
\end{align}

One thing to note here is that \\(x\_p\\) and \\(y\_p\\) are inversely proportional to \\(z\_e\\).
But the result of a multiplication between a matrix and a vector is a linear combination of its components, it's **not** possible to divide by a component.
To solve this, the hardware performs what's called a "perspective divide". Every components of the clip coordinate gets divided by its 4th component (\\(w\_c\\)), so we will set it to \\(-z\_e\\) to divide \\(x\_p\\), \\(y\_p\\) and \\(z\_p\\).

We just derived \\(w\_c\\) so we can fill one row in our projection matrix.

\\[ w\_c = -1 \times z\_e \\]

\begin{equation}
\begin{pmatrix}
. & . & . & .\\\\
. & . & . & .\\\\
. & . & . & .\\\\
0 & 0 & -1 & 0
\end{pmatrix}
\begin{pmatrix}
x\_e \\\\
y\_e \\\\
z\_e \\\\
1
\end{pmatrix}=
\begin{pmatrix}
x\_c \\\\
y\_c \\\\
z\_c \\\\
w\_c
\end{pmatrix}
\end{equation}


### Going from near plane to near clip plane {#going-from-near-plane-to-near-clip-plane}

Now that we have expressed \\(x\_p\\) and \\(y\_p\\) in terms of \\(x\_e\\) and \\(y\_e\\), let's try to express their normalized device coordinates \\(x\_n\\) and \\(y\_n\\).
The normalized device coordinates are the clip coordinates divided by their 4th component:

\begin{align}
\begin{pmatrix}
x\_n \\\\
y\_n \\\\
z\_n \\\\
w\_n
\end{pmatrix}=
\begin{pmatrix}
\frac{x\_c}{w\_c} \\\\
\frac{y\_c}{w\_c} \\\\
\frac{z\_c}{w\_c} \\\\
\frac{w\_c}{w\_c}
\end{pmatrix}
\end{align}

The near plane of our frustum is defined by 4 corners \\((l, t)\\), \\((r, t)\\), \\((r, b)\\) and \\((l, b)\\).
We want to match these with \\((-1, -1)\\), \\((1, -1)\\), \\((1, 1)\\) and \\((-1, 1)\\) respectively.

---

**<span class="underline">Note</span>**: If you have a different clip space, you will have to adjust the corners of the near clip plane!
Vulkan is different from other graphics APIs and uses a downward Y axis, and uses the same clip depth as DirectX (0 to 1).

If you use a legacy OpenGL clip space, your clip space corners will be different: Y is upward (\\(t = 1\\), \\(b = -1\\))

---

![Figure 4: The mapping of the corners of the frustum to the corners of the clip volume](/blog/persp_matrix_frustum4.svg)

<div class="figure-caption">
  
</div>

The mapping of the near frustum plane to the near clip plane is a linear function of the form \\(f(x) = \alpha x + \beta\\).
We can use the formula to find the slope of the function and then, by replacing the known values in the function, we can find the constant term:

Starting with the \\(x\\) coordinate:

\begin{align}
& f(l) = -1 \quad \text{and} \quad f( r) = 1\\\\
\Rightarrow \quad & \alpha = \frac{1 - (-1)}{r - l} = \frac{2}{r-l} \\\\
\\\\
& f( r) = 1\\\\
\Rightarrow \quad & f( r) = 1 = \frac{2}{r - l} r + \beta \\\\
\Leftrightarrow \quad &\beta = 1 - \frac{2r}{r-l} =  -\frac{r+l}{r-l}\\\\
\\\\
& f(x\_p) = x\_n\\\\
\Leftrightarrow \quad & x\_n = \frac{2}{r-l} x\_p - \frac{r+l}{r-l}
\end{align}

The same goes for finding \\(y\\):

\begin{align}
                        & f(t) = -1 \quad \text{and} \quad f(b) = 1 \\\\
\Rightarrow \quad & \alpha = \frac{1 - (-1)}{b - t} = \frac{2}{b-t} \\\\
\\\\
                            & f(b) = 1\\\\
\Rightarrow \quad     & f(b) = 1 = \frac{2}{b - t} b + \beta \\\\
\Leftrightarrow \quad &\beta = 1 - \frac{2b}{b-t} = -\frac{b+t}{b-t}\\\\
\\\\
& f(y\_p) = y\_n\\\\
\Leftrightarrow \quad & y\_n = \frac{2}{b-t} y\_p - \frac{b+t}{b-t}
\end{align}

Now we just have to replace \\(x\_p\\) and \\(y\_p\\) by the expressions we found earlier in terms of \\(x\_e\\) and \\(y\_e\\).

\begin{align}
x\_n &= \frac{2}{r-l} x\_p - \frac{r+l}{r-l}\\\\
     &= \frac{2}{r-l} \left(\frac{1}{-z\_e} n x\_e\right) - \frac{r+l}{r-l}\\\\
     &= \frac{1}{-z\_e} \left( \frac{2n}{r-l} x\_e + \frac{r+l}{r-l} z\_e \right)\\\\
\\\\
y\_n &= \frac{2}{b-t} y\_p - \frac{b+t}{b-t}\\\\
     &= \frac{2}{b-t} \left(\frac{1}{-z\_e} n y\_e\right) - \frac{b+t}{b-t}\\\\
     &= \frac{1}{-z\_e} \left( \frac{2n}{b-t} y\_e + \frac{b+t}{b-t} z\_e \right)
\end{align}

Remember that \\(x\_n = x\_c / w\_c\\) and \\(y\_n = y\_c / w\_c\\). By factoring by \\(\frac{1}{-z\_e}\\), we can read the coefficients for \\(x\_c\\) and \\(y\_c\\).

We now have:

\begin{equation}
\begin{pmatrix}
\frac{2n}{r-l} & 0 & \frac{r+l}{r-l} & 0\\\\
0 & \frac{2n}{b-t} & \frac{b+t}{b-t} & 0\\\\
. & . & . & .\\\\
0 & 0 & -1 & 0
\end{pmatrix}
\begin{pmatrix}
x\_e \\\\
y\_e \\\\
z\_e \\\\
1
\end{pmatrix}=
\begin{pmatrix}
x\_c \\\\
y\_c \\\\
z\_c \\\\
w\_c
\end{pmatrix}
\end{equation}


### Deriving the depth projection {#deriving-the-depth-projection}

Unfortunately, we cannot use the same method to find the coefficients for z, because z will always be on the near plane after projecting a point onto the near plane.
We know that the \\(z\\) coordinate does not depend on \\(x\\) and \\(y\\), so let's fill the remaining row with 0 for x and y, and A and B for the coefficients we need to find.

\begin{equation}
\begin{pmatrix}
\frac{2n}{r-l} & 0 & \frac{r+l}{r-l} & 0\\\\
0 & \frac{2n}{b-t} & \frac{b+t}{b-t} & 0\\\\
0 & 0 & A & B\\\\
0 & 0 & -1 & 0
\end{pmatrix}
\begin{pmatrix}
x\_e \\\\
y\_e \\\\
z\_e \\\\
1
\end{pmatrix}=
\begin{pmatrix}
x\_c \\\\
y\_c \\\\
z\_c \\\\
w\_c
\end{pmatrix}
\end{equation}

By definition \\(z\_n\\) is :

\\[ z\_n = \frac{z\_c}{w\_c} = \frac{A \times z\_e + B \times w\_e}{-z\_e} \\]

\\(w\_e\\) is always going to be 1.

\\[ z\_n = \frac{z\_c}{w\_c} = \frac{A \times z\_e + B}{-z\_e} \\]

To find A and B, we will replace \\(z\_n\\) and \\(z\_e\\) with known values: the near plane maps to \\(1\\) in NDC and the far plane maps to \\(0\\).

---

**<span class="underline">Note</span>**: The setup shown here is using \\(1\\) for the near plane and \\(0\\) for the far plane.
It is called "Reverse Depth" and results in a better distribution of the floating points values than using \\(-1\\) and \\(1\\) or \\(0\\) and \\(1\\) so I highly recommend to use it.

The default DirectX convention is to use \\(0\\) for near plane and \\(1\\) for far plane, legacy OpenGL uses \\(-1\\) for near and \\(1\\) for far (this is the worst for floating point precision).

---

\begin{align}
& z\_n = 1 \Rightarrow z\_e = -n\\\\
& z\_n = 0 \Rightarrow z\_e = -f
\end{align}

\begin{align}
& \left\\{
  \begin{array}{lr}
  \frac{A \times \left(-n\right) + B}{-(-n)} = 1\\\\
  \frac{A \times \left(-f\right) + B}{-(-f)} = 0
  \end{array}
  \right.\\\\
\\\\
\Leftrightarrow \quad &
  \left\\{
  \begin{array}{lr}
  A \times (-n) + B = n\\\\
  A \times (-f) + B = 0
  \end{array}
  \right.\\\\
\\\\
\Leftrightarrow \quad &
  \left\\{
  \begin{array}{lr}
  A \times (-n) + Af = n\\\\
  B = A f
  \end{array}
  \right.\\\\
\\\\
\Leftrightarrow \quad &
  \left\\{
  \begin{array}{lr}
  A = \frac{n}{f-n}\\\\
  B = \frac{nf}{f-n}
  \end{array}
  \right.
\end{align}

Here is our final expression for \\(z\_n\\):

\\[ z\_n = \frac{1}{-z\_e} \left({\frac{n}{f-n} \times z\_e + \frac{nf}{f-n}}\right) \\]

Our matrix is now complete!

\begin{equation}
\begin{pmatrix}
\frac{2n}{r-l} & 0 & \frac{r+l}{r-l} & 0\\\\
0 & \frac{2n}{b-t} & \frac{b+t}{b-t} & 0\\\\
0 & 0 & \frac{n}{f-n} & \frac{nf}{f-n}\\\\
0 & 0 & -1 & 0
\end{pmatrix}
\begin{pmatrix}
x\_e \\\\
y\_e \\\\
z\_e \\\\
1
\end{pmatrix}=
\begin{pmatrix}
x\_c \\\\
y\_c \\\\
z\_c \\\\
w\_c
\end{pmatrix}
\end{equation}


### Simplifying the matrix for the usual case {#simplifying-the-matrix-for-the-usual-case}

Most of the time we are working with symmetric frustum, that is \\(l = -r\\) and \\(b = -t\\).

The matrix becomes a bit simpler:

\begin{align}
& l = -r \Rightarrow l + r = 0 \quad \text{and} \quad  r - l = 2r = width\\\\
& b = -t \Rightarrow b + t = 0 \quad \text{and} \quad  b - t = -2t = -height
\end{align}

\begin{equation}
\begin{pmatrix}
\frac{2n}{width} & 0 & 0 & 0\\\\
0 & -\frac{2n}{height} & 0 & 0\\\\
0 & 0 & \frac{n}{f-n} & \frac{nf}{f-n}\\\\
0 & 0 & -1 & 0
\end{pmatrix}
\begin{pmatrix}
x\_e \\\\
y\_e \\\\
z\_e \\\\
1
\end{pmatrix}=
\begin{pmatrix}
x\_c \\\\
y\_c \\\\
z\_c \\\\
w\_c
\end{pmatrix}
\end{equation}

It's also easier to reason on the field of view and aspect ratios rather than on the width and the height of the near plane, so let's replace them:



![Figure 5: Diagram of the **vertical** field of view](/blog/persp_matrix_frustum5.svg)

Let's try to express \\(\frac{2n}{width}\\) and \\(-\frac{2n}{height}\\) in terms of field of view and aspect ratio:

\begin{align}
& \tan\left(\frac{fov\_y}{2}\right) = \frac{\frac{height}{2}}{n} \\\\
\Leftrightarrow \quad & 2n \tan\left(\frac{fov\_y}{2}\right) = height \\\\
\Leftrightarrow \quad & \frac{2n}{height} = \frac{1}{\tan\left(\frac{fov\_y}{2}\right)}
\end{align}

\begin{align}
\frac{2n}{width} & = \frac{2n}{width} \times \frac{height}{height} \\\\
& = \frac{2n}{height} \times \frac{height}{width} \\\\
& = \frac{2n}{height} \left(\frac{width}{height}\right)^{-1} \\\\
& = \frac{1}{\tan\left(\frac{fov\_y}{2}\right)} \left(\frac{width}{height}\right)^{-1}
\end{align}

And finally when replacing in the matrix:

\begin{align}
& \text{focal length} =  \frac{1}{\tan\left(\frac{fov\_y}{2}\right)}\\\\
\\\\
& \text{aspect ratio} =  \frac{width}{height}\\\\
\\\\
&
\begin{pmatrix}
\frac{\text{focal length}}{\text{aspect ratio}} & 0 & 0 & 0\\\\
0 & -{\text{focal length}} & 0 & 0\\\\
0 & 0 & \frac{n}{f-n} & \frac{nf}{f-n}\\\\
0 & 0 & -1 & 0
\end{pmatrix}
\begin{pmatrix}
x\_e \\\\
y\_e \\\\
z\_e \\\\
1
\end{pmatrix}=
\begin{pmatrix}
x\_c \\\\
y\_c \\\\
z\_c \\\\
w\_c
\end{pmatrix}
\end{align}


### Implementation {#implementation}

Here is a possible implementation in C++.
Note that the matrix is trivial to invert using your favourite method (I like to use row-reduction for this).
So we can compute it at the same time as the perspective projection matrix.

```cpp
float4x4 perspective(float vertical_fov, float aspect_ratio, float n, float f, float4x4 *inverse)
{
    float fov_rad = vertical_fov * 2.0f * M_PI / 360.0f;
    float focal_length = 1.0f / std::tan(fov_rad / 2.0f);

    float x  =  focal_length / aspect_ratio;
    float y  = -focal_length;
    float A  = n / (f - n);
    float B  = f * A;

    float4x4 projection({
        x,    0.0f,  0.0f, 0.0f,
        0.0f,    y,  0.0f, 0.0f,
        0.0f, 0.0f,     A,    B,
        0.0f, 0.0f, -1.0f, 0.0f,
    });

    if (inverse)
    {
        *inverse = float4x4({
            1/x,  0.0f, 0.0f,  0.0f,
            0.0f,  1/y, 0.0f,  0.0f,
            0.0f, 0.0f, 0.0f, -1.0f,
            0.0f, 0.0f,  1/B,   A/B,
        });
    }

    return projection;
}
```

A lot of people just come here to find a function to copy paste in their projects when things are not working for them, if it is not working for you I am going to try to give a few advices.

This particular implementation is using all the formulas explained in previous sections. As mentionned in the different notes, the matrix **will** change depending on your setup.
You could also be using row vectors and left-multiplication for vectors and matrix. In this case the correct matrix is the transpose of the one in this post.
If you are indeed using row vectors, I advise you to take a deep breath and make sure you understand the differences with using row vectors vs column vectors.

If you are not using the exact same setup as I do (Vulkan, Reverse depth) you **need** to derive the formulas yourself, follow what I have done with your own known values.


## Infinite perspective {#infinite-perspective}

Having to set scene-specific values for both the near and far plane can be a pain the ass.
If you want to display large open-world scenes you will almost always use an absurdly high value for the far plane anyway.

With the increased precision of using a reverse depth buffer, it is possible to set an infinitely distant far plane.

\begin{align}
& \lim\_{f \to +\infty} A = \frac{n}{f-n} = 0 \\\\
& \lim\_{f \to +\infty} B = n \times \frac{f}{f-n} = n
\end{align}

We just have to replace A with \\(0\\) and B with \\(n\\)!


## External references {#external-references}

Reverse depth
: <https://developer.nvidia.com/content/depth-precision-visualized>