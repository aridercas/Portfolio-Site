
**THE PROGRAM: KUNGFU SHADING UNITY**

**THE APPROACH:** This is a muscle-memory training ground based on the "Write first, understand later" philosophy. We are bypassing the Shader Graph to confront you directly with ShaderLab, Unity's proprietary wrapper that bridges the Inspector, the rendering pipeline, and pure HLSL code. The 20 incremental levels isolate variables, properties, and render states, forcing you to write text-based shaders and compile them in the Editor to see immediate visual results on your materials.

**THE GOAL:** By the end, you will master the anatomy of a Unity Shader. You will gain absolute technical autonomy: you will know how to expose properties to artists, command the Z-Buffer and Stencil buffer, manipulate blending operations, and structure files that efficiently compile into multiple variants using keywords, making your code ready for modern Unity rendering pipelines (URP/HDRP).

***

### Level 1: The ShaderLab Shell

SOURCE CONCEPT: "ShaderLab is Unity's shader definition language used to structure passes, render states and connect CG/HLSL code."

THE RECIPE: In Unity, create an unlit shader, open it, delete everything, and write the absolute minimum structure needed for the engine to recognize it:

```shaderlab
Shader "KungFu/Level1" {
    SubShader {
        Pass {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            float4 vert(float4 pos : POSITION) : SV_Position { return pos; }
            float4 frag() : SV_Target { return float4(1, 0, 0, 1); }
            ENDHLSL
        }
    }
}

```

THE DIAGNOSIS: Create a Material, assign the "KungFu/Level1" shader to it, and apply it to a 3D sphere. The sphere turns completely flat red.

THE "ILLUMINATION": Unity cannot read raw HLSL directly; it needs the `Shader`, `SubShader`, and `Pass` blocks to wrap the code and integrate it into its rendering pipeline.

TINKER WITH IT: Change `"KungFu/Level1"` to `"Hidden/MySecretShader"` and try to find it in the Material dropdown menu.

VISUALS: [Show a split-screen. Left: VSCode with the 10 lines of code. Right: The Unity Editor viewport showing a completely flat red sphere, with the Material inspector open].

### Level 2: The Inspector Bridge (Properties)

SOURCE CONCEPT: "The Properties block is where you declare variables... that users can adjust in the Inspector."

THE RECIPE: Insert a `Properties` block at the top, and declare the matching variable inside your `HLSLPROGRAM`:

```
Shader "KungFu/Level2" {
    Properties {
        _BaseColor ("Base Color", Color) = (1, 1, 1, 1)
    }
    SubShader {
        Pass {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            float4 _BaseColor; // Connects to Properties
            float4 vert(float4 pos : POSITION) : SV_Position { return pos; }
            float4 frag() : SV_Target { return _BaseColor; }
            ENDHLSL
        }
    }
}

```

THE DIAGNOSIS: When selecting your Material in Unity, a color picker labeled "Base Color" appears. Changing it updates the sphere's color instantly in the Scene view.

THE "ILLUMINATION": The `Properties` block creates the UI for artists, but you must explicitly declare a variable with the exact same name (`_BaseColor`) in HLSL to bridge the UI data into the GPU memory.

TINKER WITH IT: Remove `float4 _BaseColor;` from the HLSL block but leave it in `Properties`. Watch the compiler throw an undeclared identifier error.

VISUALS: [Highlight the `_BaseColor` string in the Properties block and draw a connecting line to the `float4 _BaseColor;` inside the HLSL block, showing the data bridge].

### Level 3: Numeric Restraint

SOURCE CONCEPT: "Range - A float that's constrained between a minimum and maximum value and is displayed as a slider in the Inspector."

THE RECIPE: Add a slider to control the intensity of your color:

```
    Properties {
        _BaseColor ("Base Color", Color) = (1, 0, 0, 1)
        _Intensity ("Intensity", Range(0.0, 5.0)) = 1.0
    }
// Inside HLSLPROGRAM:
float _Intensity;
float4 frag() : SV_Target { return _BaseColor * _Intensity; }

```

THE DIAGNOSIS: The Material inspector now features a slider locked between 0 and 5. Sliding it to 5 makes the red color glow brightly (if post-processing bloom is enabled).

THE "ILLUMINATION": Restricting data via `Range` prevents designers from entering values that break your mathematical lighting equations.

TINKER WITH IT: Change the range to `Range(-1.0, 1.0)` and pull the slider below zero to see negative color rendering.

VISUALS: [Screenshot of the Unity Material Inspector showing the "Intensity" slider set to 5.0, resulting in a glowing HDR color field].

### Level 4: Drawing Order (Queue)

SOURCE CONCEPT: "Queue tags determine the order in which objects are drawn by the GPU."

THE RECIPE: Tell Unity when to draw this object by adding a Tag inside your `SubShader` block:

```
    SubShader {
        Tags { "Queue"="Transparent" }
        Pass {
            // HLSL code...
        }
    }

```

THE DIAGNOSIS: Visually nothing seems to change yet, but the engine now holds this object back and renders it only after all solid walls and floors have been drawn.

THE "ILLUMINATION": The GPU is blind; `Tags` are metadata strings that sort your geometry into rendering buckets (like Background, Geometry, Transparent, Overlay) before any pixels are processed.

TINKER WITH IT: Change it to `"Queue"="Overlay"` and notice how the object suddenly draws on top of everything else in the scene, regardless of physical depth.

VISUALS: [A table showing the numerical values of the queues: Background (1000), Geometry (2000), AlphaTest (2450), Transparent (3000), Overlay (4000)].

### Level 5: Face Culling

SOURCE CONCEPT: "Cull Back (default): Only the front faces are rendered... Cull Off: Both faces are rendered."

THE RECIPE: Enter your `Pass` block and explicitly disable backface culling before the `HLSLPROGRAM` starts:

```
        Pass {
            Cull Off
            HLSLPROGRAM
            // ...

```

THE DIAGNOSIS: If you apply this material to a flat Plane or an open cylinder, you will now see the color painted on the inside/back side of the mesh, which was previously invisible.

THE "ILLUMINATION": The rasterizer discards triangles facing away from the camera to save performance (`Cull Back`), but `Cull Off` overrides this hardware optimization for thin materials like leaves or cloth.

TINKER WITH IT: Write `Cull Front`. Your object will appear inside-out, as the GPU now deletes the polygons facing you.

VISUALS: [Two planes side-by-side. One with `Cull Back` (invisible from behind) and one with `Cull Off` (visible from both sides)].

### Level 6: The Batching Buffer (CBUFFER)

SOURCE CONCEPT: "Uniform variables are per-polygon variables. Use constant buffer (cbuffer)."

THE RECIPE: Wrap your exposed HLSL properties in Unity's macro for constant buffers to optimize CPU-to-GPU communication:

```
            CBUFFER_START(UnityPerMaterial)
                float4 _BaseColor;
                float _Intensity;
            CBUFFER_END

```

THE DIAGNOSIS: The shader functions identically, but under the hood, Unity's Scriptable Render Pipeline (SRP) Batcher can now group this material with others, saving massive amounts of draw calls.

THE "ILLUMINATION": Grouping loose variables into a rigid memory block (`CBUFFER`) allows the engine to upload parameters to the GPU once per material, rather than once per object.

TINKER WITH IT: Open the Frame Debugger in Unity. Without the `CBUFFER`, objects break batches. With it, they batch perfectly.

VISUALS: [Code snippet emphasizing the `CBUFFER_START` and `CBUFFER_END` macros encapsulating the variables].

### Level 7: The Texture Request

SOURCE CONCEPT: "Textures provide image-based data for surfaces... Unity includes several texture types with built-in defaults"

THE RECIPE: Request a 2D image in `Properties`, declare it, and sample its pixels in the fragment shader:

```
    Properties {
        _MainTex ("Albedo Texture", 2D) = "white" {}
    }
// Inside HLSL:
    Texture2D _MainTex;
    SamplerState sampler_MainTex;
// Inside frag():
    return _MainTex.Sample(sampler_MainTex, float2(0,0));

```

THE DIAGNOSIS: A texture slot appears in the Inspector. The shader samples the bottom-left corner `(0,0)` of the image and paints the entire sphere with that single pixel's color.

THE "ILLUMINATION": HLSL strictly separates the raw image data (`Texture2D`) from the filtering algorithm that reads it (`SamplerState`).

TINKER WITH IT: Change `"white" {}` to `"black" {}` in the properties. Remove the texture from the inspector slot to see the default fallback apply.

VISUALS: [Material Inspector showing a Texture 2D input box with a grid pattern loaded].

### Level 8: Coordinate Mapping (UVs)

SOURCE CONCEPT: "Vector - A four-component vector. Often used to pass directional or positional data."

THE RECIPE: Extract UVs from the vertex data, pass them to the fragment, and use them to read the texture properly:

```
struct v2f {
    float4 pos : SV_Position;
    float2 uv : TEXCOORD0;
};
v2f vert(float4 pos : POSITION, float2 uv : TEXCOORD0) {
    v2f o;
    o.pos = pos;
    o.uv = uv;
    return o;
}
float4 frag(v2f i) : SV_Target {
    return _MainTex.Sample(sampler_MainTex, i.uv);
}

```

THE DIAGNOSIS: The texture wraps around the 3D object perfectly, mapping the 2D image coordinates onto the 3D surface.

THE "ILLUMINATION": UVs are simply numerical coordinates ranging from `0` to `1` that tell the sampler exactly where to look on the 2D image for every pixel on the screen.

TINKER WITH IT: Multiply `i.uv * 2.0` inside the `Sample` function to make the texture tile twice across the geometry.

VISUALS: [A 3D cube with a checkerboard texture properly unwrapped onto its faces].

### Level 9: Attribute Drawers

SOURCE CONCEPT: "[NoScaleOffset] - Prevents the display of tiling and offset fields for texture properties."

THE RECIPE: Clean up your Inspector by adding an Attribute Drawer directly above your texture property:

```
    Properties {
        [NoScaleOffset] _MainTex ("Albedo Texture", 2D) = "white" {}
    }

```

THE DIAGNOSIS: The Tiling (X, Y) and Offset (X, Y) input boxes disappear from the Material Inspector next to your texture.

THE "ILLUMINATION": Brackets `[ ]` in ShaderLab are UI commands for the Unity Editor, altering how data is presented without changing a single line of the compiled GPU code.

TINKER WITH IT: Replace `[NoScaleOffset]` with `[Normal]` to force the editor to warn users if they drop a non-normal map into the slot.

VISUALS: [Before-and-after comparison of the Material Inspector: one with Tiling/Offset boxes, and one clean without them due to `[NoScaleOffset]`].

### Level 10: State Toggles

SOURCE CONCEPT: "[Toggle(ENABLE_FANCY)] - Simulates a boolean toggle... and auto-generates a corresponding shader keyword"

THE RECIPE: Add a checkbox to disable the texture and return pure white:

```
    Properties {
        [Toggle(USE_TEXTURE)] _UseTex ("Enable Texture", Float) = 1
    }
// Inside HLSLPROGRAM:
    #pragma shader_feature USE_TEXTURE
// Inside frag():
    #if defined(USE_TEXTURE)
        return _MainTex.Sample(sampler_MainTex, i.uv);
    #else
        return float4(1,1,1,1);
    #endif

```

THE DIAGNOSIS: A checkbox named "Enable Texture" appears in the Inspector. Toggling it swaps the material instantly between the texture and a solid white color.

THE "ILLUMINATION": GPUs hate `if/else` statements. The `[Toggle]` attribute commands Unity to compile two separate physical versions of your shader (variants) and swap them on the fly.

TINKER WITH IT: Uncheck the box, then look at the compiled code statistics in the inspector to see the instruction count drop.

VISUALS: [Material Inspector showing the Checkbox active, alongside a diagram showing the `#ifdef` branching logic].

### Level 11: The Blending Equation

SOURCE CONCEPT: "Blending combines the fragment shader's output color (SrcValue) with the color already in the render target (DstValue)."

THE RECIPE: In your `Pass` block, inject the classic alpha blending command to combine pixels:

```
        Tags { "Queue"="Transparent" }
        Pass {
            Blend SrcAlpha OneMinusSrcAlpha
            ZWrite Off
            // HLSL...
            return float4(1, 0, 0, 0.5); // 50% opacity red
        }

```

THE DIAGNOSIS: The red sphere becomes semi-transparent, allowing background objects to be seen through it at a 50% mix.

THE "ILLUMINATION": `Blend` grabs the color your shader just calculated (`Src`) and mathematically multiplies it against the color already on your monitor (`Dst`).

TINKER WITH IT: Change it to `Blend One One` (Additive Blending) to make the object look like a glowing hologram that adds light to the scene.

VISUALS: [A table illustrating the Blend math: `(Source * SrcAlpha) + (Destination * OneMinusSrcAlpha)`].

### Level 12: Z-Buffer Pacifism

SOURCE CONCEPT: "ZWrite controls whether a shader writes depth information... Typically On for opaque objects and Off for transparent ones to avoid Z-fighting."

THE RECIPE: Verify the `ZWrite Off` command you added in the previous level.

THE DIAGNOSIS: The transparent sphere does not cast "invisible" physical depth into the scene. Other transparent objects drawn behind it will still be visible.

THE "ILLUMINATION": If a transparent object writes to the depth buffer (`ZWrite On`), the GPU will assume it's a solid wall and prematurely cull anything behind it, causing severe visual artifacts.

TINKER WITH IT: Delete `ZWrite Off` (which defaults it to On) and place another transparent object behind this one. Watch the background object disappear entirely.

VISUALS: [Diagram showing the Z-Buffer as a grayscale depth map, with the transparent object leaving no footprint when `ZWrite Off` is active].

### Level 13: X-Ray Vision (ZTest)

SOURCE CONCEPT: "ZTest defines how depth testing is performed by comparing each pixel's depth to the Z-Buffer."

THE RECIPE: Inside your `Pass`, command the depth test to ignore physical rules:

```
        Pass {
            ZTest Always
            // HLSL...

```

THE DIAGNOSIS: Your object renders perfectly visibly on top of the screen, even if it is physically placed behind a thick, opaque brick wall in the scene.

THE "ILLUMINATION": The Z-Test is just a conditional `<` or `>` check. By overriding it to `Always`, the GPU prints the pixel regardless of what the depth buffer claims is in front of it.

TINKER WITH IT: Change it to `ZTest Greater`. The object will only render when it is _hidden_ behind another object (perfect for occlusion outlines).

VISUALS: [An image of a character hidden behind a wall, but rendering entirely over it due to `ZTest Always`].

### Level 14: The Binary Cutout

SOURCE CONCEPT: "AlphaToMask forces the shader to treat the alpha channel as a binary mask rather than a continuous gradient."

THE RECIPE: Render a soft alpha gradient on a solid object, but force the hardware to cut it sharply:

```
        Pass {
            AlphaToMask On
            // HLSL...
            return float4(1, 0, 0, i.uv.x); // Gradient alpha
        }

```

THE DIAGNOSIS: Instead of a smooth fade to invisibility, the object will have a harsh, jagged edge exactly halfway across its geometry.

THE "ILLUMINATION": `AlphaToMask` (Alpha-to-Coverage) interacts directly with Multisample Anti-Aliasing (MSAA) at the hardware level to drop pixels completely, maintaining correct Z-sorting for foliage and fences without expensive transparency sorting.

TINKER WITH IT: Turn `AlphaToMask Off` and watch the gradient return, but break depth sorting if `ZWrite` is still on.

VISUALS: [A plane showing a smooth gradient alpha vs. a jagged binary cutoff on the right half].

### Level 15: Channel Isolation

SOURCE CONCEPT: "The ColorMask command allows you to restrict the GPU to writing only specific color channels (Red, Green, Blue, or Alpha)."

THE RECIPE: Block the GPU from writing green and blue colors to the screen:

```
        Pass {
            ColorMask R
            // HLSL...
            return float4(1, 1, 1, 1); // Trying to return white
        }

```

THE DIAGNOSIS: Even though the HLSL code explicitly tells the GPU to render pure white `(1,1,1,1)`, the object on screen will be pure red.

THE "ILLUMINATION": `ColorMask` puts a physical lock on the frame buffer's memory channels before the pixel is written, saving bandwidth for specific rendering passes (like shadow maps).

TINKER WITH IT: Write `ColorMask 0`. The object will become completely invisible, but it will still write to the Z-Buffer (creating an invisible cloak that occludes other geometry).

VISUALS: [A padlock icon over the G, B, and A channels, letting only the R channel pass into the monitor].

### Level 16: Modern Space Transformation

SOURCE CONCEPT: "URP: TransformObjectToHClip( positionOS.xyz )"

THE RECIPE: Ditch the manual matrix multiplication for Unity's URP library functions. Add an include and update the vertex shader:

```
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            float4 vert(float3 pos : POSITION) : SV_Position {
                return TransformObjectToHClip(pos);
            }

```

THE DIAGNOSIS: The object projects onto the camera exactly as it did before, but the code is shorter and explicitly compatible with Universal Render Pipeline requirements.

THE "ILLUMINATION": `Core.hlsl` houses Unity's standardized math matrices. Using `TransformObjectToHClip` guarantees that your geometry scales and rotates safely across all platforms (VR, Desktop, Mobile).

TINKER WITH IT: Try to compile this without the `#include`. The compiler will crash, missing the function definition.

VISUALS: [Code block highlighting the `#include` path to the URP Core library].

### Level 17: Front vs. Back Sensibility

SOURCE CONCEPT: "SV_IsFrontFace... allows us to project different colors and textures on both mesh faces."

THE RECIPE: Read the hardware face direction and use it to branch colors. Remember to turn `Cull Off` first.

```
        Cull Off
// ...
        float4 frag(v2f i, bool isFront : SV_IsFrontFace) : SV_Target {
            return isFront ? float4(0,1,0,1) : float4(1,0,0,1);
        }

```

THE DIAGNOSIS: If you look at the outside of your mesh, it is green. If you fly the camera inside the mesh, the interior walls are red.

THE "ILLUMINATION": The rasterization hardware automatically flags whether the triangle's vertices were drawn clockwise or counter-clockwise, injecting this boolean (`SV_IsFrontFace`) into the fragment shader for free.

TINKER WITH IT: Reverse the condition: `isFront ? float4(1,0,0,1) : float4(0,1,0,1)`.

VISUALS: [A sphere sliced in half. The outer shell is green, the hollow interior cavity is painted red].

### Level 18: The Invisible Ink (Stencil Write)

SOURCE CONCEPT: "The Stencil Buffer is a specialized buffer that stores an 8-bit integer (values 0–255) for each pixel... The GPU can perform a Stencil Test"

THE RECIPE: Create a new shader for a "Mask" object that draws nothing but writes the number `2` to the Stencil buffer:

```
    SubShader {
        Tags { "Queue"="Geometry-1" }
        ColorMask 0
        ZWrite Off
        Stencil {
            Ref 2
            Comp Always
            Pass Replace
        }
        Pass { // Empty HLSL }
    }

```

THE DIAGNOSIS: Applied to a plane, the plane becomes completely invisible. However, it secretly stamped the number `2` on the monitor's hidden Stencil Buffer wherever it was located.

THE "ILLUMINATION": You can manipulate pixels without touching color. This shader acts as an invisible die-stamp, preparing the screen for a subsequent rendering pass.

TINKER WITH IT: Change `ColorMask 0` to `ColorMask RGBA` to see the mask shape physically render on the screen.

VISUALS: [A conceptual diagram showing a "Color Buffer" (empty) layered over a "Stencil Buffer" (showing a white square stamped with a '2')].

### Level 19: The Portal Window (Stencil Read)

SOURCE CONCEPT: "It compares the current Stencil Buffer value with the reference value (2) and uses a comparison function to decide whether to draw a pixel."

THE RECIPE: On your main `KungFu/Level2` object shader, instruct it to only render if it finds the number `2`:

```
    SubShader {
        Stencil {
            Ref 2
            Comp Equal
        }
        Pass {
            // HLSL...

```

THE DIAGNOSIS: Your object is completely invisible _unless_ it is physically behind the invisible mask plane you created in Level 18. You have created a magic window or portal.

THE "ILLUMINATION": `Comp Equal` checks the 8-bit memory of the Stencil Buffer before running the fragment shader. If the number isn't `2`, the GPU aborts the pixel, creating non-euclidean visual tricks.

TINKER WITH IT: Change it to `Comp NotEqual`. Now the object exists everywhere _except_ inside the mask window, punching a hole through it.

VISUALS: [A 3D cube only partially visible because it is being viewed through a 2D quad acting as a magic stencil portal].

### Level 20: Preprocessor Engineering (Multi_Compile)

SOURCE CONCEPT: "multi_compile: Unity generates shader variants for every possible combination of these keywords."

THE RECIPE: Command Unity to build multiple permanent versions of this shader using a global macro:

```
        Pass {
            HLSLPROGRAM
            #pragma multi_compile _ MAGIC_MODE

            float4 frag() : SV_Target {
                #if defined(MAGIC_MODE)
                    return float4(1, 0, 1, 1); // Magenta
                #else
                    return float4(0, 0, 0, 1); // Black
                #endif
            }
            ENDHLSL
        }

```

THE DIAGNOSIS: The object renders black. If a C# script calls `Shader.EnableKeyword("MAGIC_MODE");`, the object instantly turns magenta without evaluating a single `if` statement on the GPU.

THE "ILLUMINATION": `#pragma multi_compile` forces the Unity offline compiler to slice your text document into multiple raw binary files. Swapping keywords at runtime swaps the physical binary, ensuring peak execution speed.

TINKER WITH IT: Use `shader_feature` instead of `multi_compile`. Note how the variant is stripped from the final build if no material uses it in the editor.

VISUALS: [Flowchart showing `Shader.hlsl` splitting into two separate `.bin` files (`Normal` and `MAGIC_MODE`) during the build process].

----------

**Navigation Index**

- Level 1: The ShaderLab Shell
- Level 2: The Inspector Bridge (Properties)
- Level 3: Numeric Restraint
- Level 4: Drawing Order (Queue)
- Level 5: Face Culling
- Level 6: The Batching Buffer (CBUFFER)
- Level 7: The Texture Request
- Level 8: Coordinate Mapping (UVs)
- Level 9: Attribute Drawers
- Level 10: State Toggles
- Level 11: The Blending Equation
- Level 12: Z-Buffer Pacifism
- Level 13: X-Ray Vision (ZTest)
- Level 14: The Binary Cutout
- Level 15: Channel Isolation
- Level 16: Modern Space Transformation
- Level 17: Front vs. Back Sensibility
- Level 18: The Invisible Ink (Stencil Write)
- Level 19: The Portal Window (Stencil Read)
- Level 20: Preprocessor Engineering (Multi_Compile)
