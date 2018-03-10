---
layout: post
title:  "Kepler's Third Law in Schwarzschild"
date:   2015-12-04 21:36:01
categories: physics relativity black hole Kepler
comments: True
permalink: posts/keplers-law-schwarzschild
shortUrl: http://goo.gl/ZDO831
is_post: true
image: https://upload.wikimedia.org/wikipedia/commons/1/1d/Angular_Parameters_of_Elliptical_Orbit.png
description: >
             Kepler's third law states that the square of the orbital
             period of a particle is proportional to the cube of the
             semi-major axis of its orbit. In this statement, does the
             time coordinate correspond to the Schwarzschild
             time or the proper time?
---
Kepler's third law in classical
mechanics states that the square of the orbital period, $$T$$, of a
planet or particle is proportional to the cube of the semi-major axis
of its orbit. The classical equation describing this is

$$ \left(\frac{d\phi}{dt_{\text{Newt}}}\right)^2=\frac{G(M+m)}{a^3} $$

where $$a$$ is the semi-major axis, $$t_{\text{Newt}}$$ the Newtonian
time, and $$m$$ the mass of the test particle. We are interested in
test particles, so $$M+m\approx M$$.

An interesting question is how this generalizes to test particles
moving around a very massive body in general relativity, such as a
black hole. The metric in Schwarzschild coordinates that describes
this geometry is

$$ ds^2=-(1-2GM/r)dt^2+\frac{dr^2}{1-2GM/r}+r^2d\Omega^2$$

where $$M$$ is the mass of the black hole. The question is whether
Kepler's third law applies to the Schwarzschild time, $$t$$, or the
proper time, $$\tau$$, of the test particle?
To answer this question let us assume that
$$\theta=\pi/2$$ and that the motion is circular, so
$$\frac{dr}{d\tau}=\frac{d^2r}{d\tau^2}=0$$.

The radial geodesic equation is given by (see, for example, [Sean
Carroll][1], or [Misner, Thorne, and Wheeler][2])

$$\frac{d^2r}{d\tau^2}+\frac{GM}{r^3}(r-G2M)\left(\frac{dt}{d\tau}\right)^2-\frac{GM}{r(r-2GM)}\left(\frac{dr}{d\tau}\right)^2-(r-2GM)\left[\left(\frac{d\theta}{d\tau}\right)^2+\sin^2(\theta)\left(\frac{d\phi}{d\tau}\right)^2\right]=0$$

Using our assumptions this simplifies to

$$\frac{GM}{r^3}\left(\frac{dt}{d\tau}\right)^2-\left(\frac{d\phi}{d\tau}\right)^2=0$$

From this we see that

$$\left(\frac{d\phi}{dt}\right)^2=\frac{GM}{r^3}$$

and that it is the Schwarzschild time coordinate that is analogous to
the Newtonian time in Kepler's third law.

[1]: http://www.amazon.com/Spacetime-Geometry-Introduction-General-Relativity/dp/0805387323/ref=sr_1_4?ie=UTF8&qid=1449282794&sr=8-4&keywords=sean+carroll
[2]: http://www.amazon.com/Gravitation-Physics-Charles-W-Misner/dp/0716703440/ref=sr_1_1?ie=UTF8&qid=1449282855&sr=8-1&keywords=misner+thorne+wheeler
