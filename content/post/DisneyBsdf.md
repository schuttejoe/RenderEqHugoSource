---
title: "Implementing the Disney BSDF for the Moana Island Scene"
date: 2018-08-10T12:04:59-07:00
preview: "img/Posts/DisneyBsdf/Preview.png"
description: "A write-up showing a full implementation of the Disney BSDF"
draft: false
---

$\newcommand{\i}{\\textit{i}}$
$\newcommand{\o}{\\textit{o}}$
$\newcommand{\n}{\\textit{n}}$
$\newcommand{\h}{\\textit{h}}$
$\newcommand{\v}{\\textit{v}}$
$\newcommand{\x}{\\textit{x}}$
$\newcommand{\y}{\\textit{y}}$
$\newcommand{\ht}{\\textit{h}\_t}$

With the release by Disney of their Moana Island scene my interest in supporting the Disney BSDF in my renderer has gone from "I'll definitely get to it at some point" to "Do it! Do it! COME ON, DO IT NOW!". Furthermore, it's been a while since I've posted anything here so I figured I'd do a writeup about it. Also, these blog posts always keep me honest and force me to make sure I'm getting the details right so we all know who is __really__ the main benefactor here :P.

The Disney BRDF was introduced by Brent Burley as part of the [SIGGRAPH 2012 Physically Based Shading course](http://blog.selfshadow.com/publications/s2012-shading-course/) and then extended to a BSDF with integrated subsurface scattering for the [SIGGRAPH 2015 Physically Based Shading course](http://blog.selfshadow.com/publications/s2015-shading-course/). While the model is not strictly physically based it was designed with physically based principles in mind and each of the terms included was done so to fit the model to materials in the MERL 100 database. More importantly, each term was carefully chosen and parameterized to be easy for artists to use and so that blending multiple layers together would give intuitive results. I believe it is these later traits that have made the Disney BSDF to be so successful.

The materials in the Moana Island scene are all one of two types: "solid" or "thin". Both material types follow the same model and are only split into two for practical, production reasons. For thin materials there is no internal surface to apply subsurface scattering with so it is approximated with Hanrahan-Krueger diffuse. I'll dig more into that later. Additionally, since there is also no volume to refract into and later out of both events are modeled simultaneously at the same point. The most common example of this material type in the Island scene is leaves and other foliage where one quad could model one or even several leaves. The solid material type would then make sense for objects that are modeled as more than just a flat polygon.

The title of my blog wouldn't be a bad pun for giving you a bunch of equations if I didn't include a whole bunch of equations in each post. So next I'll describe each individual lobe of the Disney BSDF then we'll wrap things up by showing you how those terms are combined for use with the thin and solid material types.

---
Sheen
---

The sheen lobe is probably the simplest of all of the terms so we'll start here. It is a standalone lobe that is added to the others based on a $\textit{sheen}$ parameter with its color varying between white and the base color depending on the $\textit{sheenTint}$ parameter. It's purpose is to model light at grazing angles on a surface so it will mostly be used for retro-reflection seen in cloth or on rougher surfaces to add back in energy lost due to a geometry term that only models single-scattering. I wouldn't be surprised to see this term be replaced with something like [Multiple-Scattering Microfacet BSDFs with the Smith Model](https://eheitzresearch.wordpress.com/240-2/) or [A Multi-Faceted Exploration (Part 2)](http://blog.selfshadow.com/2018/06/04/multi-faceted-part-2/) in Disney's future iterations though that would necessitate an alternate solution for retro-reflection.

While I don't recall seeing it mentioned anywhere in the notes, in the [Disney BRDF Explorer](https://github.com/wdas/brdf) they extract the hue and saturation from the base color and use that for the sheen lobe's tint color rather than using the base lobe directly. This is done by calculating the luminance using approximate, linear-space CIE luminance weights and then normalizing the luminance:
~~~
float luminance = Dot(float3(0.3f, 0.6f, 1.0f), baseColor);
float3 tint = (luminance > 0.0f) ? baseColor * (1.0f / luminance) : float3::One_;
~~~

With that we can then calculate the intensity of the sheen lobe:
$$f(\textit{sheen}, \theta_d) = \textit{sheen} * ((1 - \textit{sheenTint}) + \textit{sheenTint} * \textit{tint}) * (1 - \cos\theta_d)^5$$

~~~~

~~~~

---
Clearcoat
---

The Clearcoat lobe is another additive lobe that is controlled by a $\textit{clearcoat}$ parameter for its intensity and a $\textit{clearcoatGloss}$ parameter for its shape. This lobe is a bit more complicated and models a full BRDF but most of its terms are fixed to keep this lobe as a simple, artist-friendly way to model, well, a clear... coating on top of a material. It's pretty well named.

The distribution term used for the clearcoat layer was created by Buris called Generalized Trowbridge-Reitz with a fixed gamma of 1 to give it a long tail. Here is its normalized form via Burley's course notes:

$$D\_{GTR\_1}(x) = \frac{\alpha^2 - 1}{\pi\log(\alpha^2)} \frac{1}{1 + (\alpha^2 - 1)\cos^2\theta\_h}$$

The Fresnel term uses the Schlick approximation with a fixed index of refraction of 1.5 to be representative of polyurethane. This evaluates to $F\_0 = 0.04$.

$$f\_{Schlick} = F\_0 + (1 - F\_0)(1 - \cos\theta\_h)^5$$

The masking-shadowing term used was the separable form of Smith for GGX (aka GTR2) with a fixed roughness of 0.25. While the term is not the correct match for the distribution of normals as of the 2014 addendum to the 2012 course notes it looks like they were happy with the look and have not changed it.

$$G(\theta, \alpha) = \frac{1}{\cos\theta + \sqrt{\alpha^2 + \cos\theta - \alpha^2\cos^2\theta}}$$
$$G(\theta\_l, \theta\_v) = G(\theta\_l, 0.25) * G(\theta\_v, 0.25)$$

~~~~

~~~~

---
Specular BRDF
---

The specular BRDF is the traditional Cook-Torrance microfacet BRDF that uses Anisotropic GGX (aka GTR2) with the Schlick Fresnel approximation and the Smith masking-shadowing function. Burley calculated the anisotropic weights $\alpha\_x$ and $\alpha\_y$ with:

$$\textit{aspect} = \sqrt{1 - 0.9 * \textit{anisotropic}}$$
$$\alpha\_x = \textit{roughness}^2 / \textit{aspect}$$
$$\alpha\_y = \textit{roughness}^2 * \textit{aspect}$$

The anisotropic distribution of normals is:

$$D\_{GTR\_2aniso} = \frac{1}{\pi \alpha\_x \alpha\_y} \frac{1}{(\frac{(\h \cdot \x)^2}{\alpha\_x^2} + \frac{(\h \cdot \y)^2}{\alpha\_y^2} + (\h \cdot \n)^2)^2}$$

Via the 2014 addendum to the course notes we know that Disney changed their geometry term to match the anisotropic Smith GGX term derived by Heitz in [__Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs__](http://jcgt.org/published/0003/02/03/).

$$G1\_{GTR\_2aniso} = \frac{1}{(\v \cdot \n) + \sqrt{(\alpha\_x  \v \cdot \x)^2 + (\alpha\_y  \v \cdot \y)^2 + (\v \cdot \n)^2}}$$

The Fresnel term here gets somewhat complicated due to the versatility of this lobe. The <i>specularTint</i> parameter was added to provide artistic control and is used to tint the specular color from white towards the same __tint__ I described in the Sheen section. 

~~~~

~~~~

---
Specular BSDF
---

The specular BSDF is an extension of the specular BRDF to handle refraction. Since I haven't discussed refraction in this blog I'll note that the microfacet form for it is given by:

$$f\_t(\i,\o,\n) = \frac{|\i \cdot \ht|}{|\i \cdot \n|}\frac{|o \cdot \ht|}{|\o \cdot \n|}\frac{\eta^2}{(\i \cdot \ht + \eta\o\cdot\ht)^2}\frac{1}{\eta^2} (1 - F(\i, \ht)) G(\i,\o,\ht) D(\ht)$$

with $\ht = -\frac{\i + \eta\o}{||i + \eta\o||}$ and $\eta = \frac{\eta\_i}{\eta\_o}$ being the relative index of refraction.

~~~

~~~

---
Diffuse BRDF
---

The dielectric BRDF is quite a bit more complicated than the typical Lambert diffuse model. Theirs includes both a diffuse Fresnel factor as well as a term for diffuse retro-reflection. Additionally, the non-directional portion of the model is split out such that it can be replaced with a subsurface model using either diffusion or volumetric scattering. The whole diffuse lobe is given as:

$$f\_d = f\_{Lambert} (1 - 0.5F\_L)(1 - 0.5F\_V) + f\_{retro-reflection}$$

with:

$$f\_{retro-reflection} = \frac{baseColor}{\pi} R\_R(F\_L + F\_V + F\_LF\_V(R\_R - 1))$$
$$F\_L = (1 - \cos\theta\_l)^5$$
$$F\_v = (1 - \cos\theta\_v)^5$$
$$R\_R = 2 * \textit{roughness} * \cos^2(\theta\_d)$$

and $f\_{Lambert}$ being either a simple Lambert term:

$$f\_{Lambert} = \frac{\textit{baseColor}}{\pi}$$

---
Thin model
---

---
Solid model
---


---

Thank you so much for reading. Please leave any feedback here: [__@schuttejoe__](https://twitter.com/schuttejoe)!

---

References:

* [__Disney's Moana Island Scene__](https://www.disneyanimation.com/technology/datasets)
* [__Physically Based Shading at Disney__ - Burley 2012](http://blog.selfshadow.com/publications/s2012-shading-course/burley/s2012_pbs_disney_brdf_notes_v3.pdf)
* [__Extending the Disney BRDF to a BSDF with Integrated Subsurface Scattering__ - Burley 2015](http://blog.selfshadow.com/publications/s2015-shading-course/burley/s2015_pbs_disney_bsdf_notes.pdf)
* [__SIGGRAPH 2012 Course: Practical Physically Based Shading in Film and Game Production__](http://blog.selfshadow.com/publications/s2012-shading-course/)
* [__SIGGRAPH 2015 Course: Physically Based Shading in Theory and Practice__](http://blog.selfshadow.com/publications/s2015-shading-course/)
* [__Disney BRDF Explorer__](https://github.com/wdas/brdf)
* [__Microfacet Models for Refraction through Rough Surfaces__](https://www.cs.cornell.edu/~srm/publications/EGSR07-btdf.html)
* [__Memo on the Fresnel Equations__](https://seblagarde.wordpress.com/2013/04/29/memo-on-fresnel-equations/)
* [__Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs__](http://jcgt.org/published/0003/02/03/)
* [__Practical and Controllable Subsurface Scattering for Production Path Tracing__](https://disney-animation.s3.amazonaws.com/uploads/production/publication_asset/153/asset/siggraph2016SSS.pdf)

Additional Thanks:

* [Morgan McGuire for the cleaned up version of Stanford Graphics Lab's Chinese Dragon model](http://casual-effects.com/data/)
* [CC0 Textures for the paper texture](https://cc0textures.com)
