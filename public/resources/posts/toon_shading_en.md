# Creating a Toon Shader: Adding Volume with Specular and Rim Light

![Final render of the Toon Shader with Specular and Rim Light](resources/videos/ToonShading_Lights.gif)

In the previous step of my Technical Art roadmap, I presented how to transform a classic light gradient into sharp flat areas of color (Cell Shading). However, flat colors are not enough to define an object's volume or detach it from its environment.

Today, I’m going further by adding two other essentials of lighting models: **Specular Light** (bright reflections) and **Rim Light** (the contour halo).

---

## The two silhouette light sources

For a stylized object to have more depth, it must react to light in a specific way:
1.  **Specular Light:** This is the direct reflection of the light source. In an NPR style, instead of being a blurry gradient, it usually becomes a sharp spot that emphasizes the object's curvature.
2.  **Rim Light:** This is a light halo that runs along the edges of the silhouette. It is crucial for detaching a character from a dark background.

---

## Differences between Unity and Unreal Engine

As with the previous chapter, the two engines take radically different approaches.

*   **For Unity:** I kept the same approach with direct access to data via the `Main Light`.
*   **For Unreal:** I previously used a global Post-Process. While ideal for flattening the entire image, it is very limited for Specular and Rim Light, as these effects depend on a light's precise direction and the camera's position.

I decided to use a **MPC (Material Parameter Collection)** this time. Unlike a Post-Process, which acts like a filter on the screen, an MPC allows you to send information (such as the sun's position) directly to all the materials in the game. This allows each object to calculate its own specular highlight accurately while remaining synchronized with the rest of the world.

### MPC Setup:

Here is a brief overview of how to set up a **Material Parameter Collection (MPC)**:

1.  In the Content Browser, I create an MPC. Inside, I need a **Vector** parameter, which I named `SunDirection`.
2.  To make the direction dynamic, the rotation of the `Directional Light` must be linked to the MPC parameter. To do this, in the **Level Blueprint**, I retrieved the light's *Forward Vector* and used the `Set Vector Parameter Value` node to update the MPC every frame.
3.  Finally, in the Material Graph, I call the `CollectionParameter` node and select the MPC along with the `SunDirection` parameter. This provides a usable vector for Dot Product calculations, exactly like in Unity!

![MPC Content in Unreal](resources/visuels/posts/toonshading/ue_mpc.png)

![Material Graph configuration with the MPC](resources/visuels/posts/toonshading/ue_nodes_mpc_setup.png)

---

## Specular Light: The Blinn-Phong Model

Calculating reflections is relatively complicated by default. To simplify it—both for understanding and project performance—I use the **Blinn-Phong** model. It is an optimized version of the classic Phong model, ideal for real-time rendering.

> **Note:** Other more complex models exist, such as **Oren-Nayar** (for very rough surfaces like skin) or **Cook-Torrance** (used in PBR for extreme realism). But for Toon Shading, Blinn-Phong is the most suitable due to its simplicity.

Here is how to implement specular light:

1.  Unlike diffuse light, which remains the same regardless of where you look at the object, specular light depends on the **camera position**. If you move, the reflection moves on the surface. You must therefore use the **Camera Vector (V)**.
2.  In the original Phong model, you had to calculate the light's exact reflection vector, which is mathematically expensive as it involves complex symmetrical vector calculations. Blinn-Phong simplifies this with the **Half-Vector (H)**: it is the normalized average between the Light direction (L) and the View direction (V). If the mesh normal is perfectly aligned with this "middle vector," the reflection is at its maximum intensity. Without it, the reflection calculation is much heavier for the GPU.
3.  Finally, to achieve that "sharp spot" effect, we pass the result into a `step` or `smoothstep` function.

```hlsl
// Calculate the Half-Vector
float3 halfVector = normalize(lightDir + viewDir);

// Dot product between the normal and the half-vector
float dotNH = saturate(dot(normal, halfVector));

// Transformation into a sharp spot (Toon)
float specIntensity = smoothstep(_SpecSize - _SpecSmooth, _SpecSize + _SpecSmooth, dotNH);
float3 specularColor = specIntensity * _SpecularTint.rgb * mainLight.color;
```

For Unreal, as mentioned earlier, we use an **MPC** to retrieve the sun's direction and perform the calculation. The result is applied as an **emissive color** to override the post-process and force the Specular light (and subsequently the Rim light) to be drawn on top of the Cell Shading.

![Unreal nodes logic for specular light](resources/visuels/posts/toonshading/ue_nodes_specular_lighting.png)

![Specular Light Render](resources/visuels/posts/toonshading/specular_lighting.png)

---

## Rim Light: The Fresnel Effect

As a reminder, Rim Light (or contour light) simulates light coming from behind the object and "bleeding" over the edges. Consequently, it once again depends on the view (camera position) and, of course, the mesh normal.

As usual, this relies on a dot product—this time between the **Normal (N)** and the **View Vector (V)**. We interpret the result (ranging between [-1, 1]) as follows:

*   At the center of the object, the vectors are aligned (Dot = 1).
*   At the edges, the vectors are perpendicular (Dot = 0).

If left as is, the dot product would produce white at the center. To obtain a contour, we must invert the result. By calculating `1.0 - dot(N, V)`, we get a value of **1 at the edges** (bright) and **0 at the center** (black). This is known as the **Fresnel effect**.

Here is the HLSL code:

```hlsl
// Calculate the inverted Fresnel
float rimDot = 1.0 - saturate(dot(normal, viewDir));

// Sharp threshold for the Toon effect
float rimIntensity = smoothstep(_RimThreshold - _RimSmooth, _RimThreshold + _RimSmooth, rimDot);

// Light masking (optional) so the Rim only appears on the lit side
rimIntensity *= saturate(dot(normal, lightDir));

float3 rimColor = rimIntensity * _RimColor.rgb;
```

In Unreal, you can use the native **Fresnel** node. In my case, I used the same solution as in Unity with a simple `Dot` followed by a `OneMinus`.

![Unreal nodes logic for Rim Light](resources/visuels/posts/toonshading/ue_nodes_rim_lighting.png)

![Rim Light Render](resources/visuels/posts/toonshading/rim_lighting.png)

> **Note:** For high-quality rendering, you can often blend the mesh normals (**Vertex Normals**) with the texture normals (**Pixel/Normal Map**). This allows for a Rim Light that is either perfectly smooth (clean manga style) or highly detailed and rugged, depending on your artistic needs.

---

## Conclusion

By combining **Diffuse**, **Ambient**, **Specular**, and **Rim Light**, you achieve a complete lighting render that defines volumes effectively while maintaining an NPR look.

![Complete final render](resources/videos/ToonShading.gif)