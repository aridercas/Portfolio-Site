
# KUNGFU SHADING UNREAL ENGINE 5

**THE APPROACH:** This is a muscle-memory training ground based on the "Write first, understand later" philosophy. We will tackle the Unreal Engine 5 Material Editor architecture head-on. The 20 incremental levels isolate variables and hardware rules, forcing you to execute node-based logic and compile to see immediate visual results. Throughout the program, you will explore everything from basic physical inputs to the manipulation of Static Switches, channel packing, and pure HLSL code injection, culminating in mathematical optimization per vertex.

**THE GOAL:** By the end, you will master the "black box" of the UE5 material architecture. You will understand exactly how visual nodes collapse into silicon instructions. You will gain absolute technical autonomy: you will know how to expose clean interfaces (Material Instances) for artists, how to avoid catastrophic "shader explosions", how to hack `structs` directly into the graph, and how to optimize heavy calculations by moving them out of the pixel shader.

***

### Level 1: The Main Material Node

SOURCE CONCEPT: "Materials tell the engine exactly how a surface should interact with the light... The inputs on the Main Material Node are the last stop in a UE5 Material graph".

THE RECIPE: Open the Material Editor. Hold the `3` key and Left-Click to spawn a `Constant3Vector` node. Double-click its color swatch and set it to pure red (R=1, G=0, B=0). Connect the output pin to the `Base Color` input on the Main Material Node.

THE DIAGNOSIS: The preview sphere in the top-left corner instantly becomes red, but looks completely flat, like unpolished clay.

THE "ILLUMINATION": The Main Material Node receives vector math and automatically routes it into the compiled HLSL pixel shader output of the engine.

TINKER WITH IT: Change the color to blue and observe the real-time update in the preview window.

VISUALS: [Screenshot of the Material Editor focusing on a Constant3Vector connected via a white wire to the Base Color pin of the Main Material Node].

### Level 2: Physical Constraints

SOURCE CONCEPT: "Metallic defaults to 0 (non-metallic)... Specular defaults to 0.5... Roughness defaults to 0.5.".

THE RECIPE: Hold the `1` key and Left-Click to spawn a 1D `Constant` node. Type `0.07` in its value field. Connect its output to the `Roughness` input of the Main Material Node.

THE DIAGNOSIS: The sphere stops looking like matte clay and suddenly reflects its environment sharply, mimicking polished plastic.

THE "ILLUMINATION": Physically Based Rendering (PBR) values are driven by simple 1D floating-point numbers (from 0.0 to 1.0) that dictate how light scatters across the surface micro-facets.

TINKER WITH IT: Create another Constant set to `1.0` and connect it to `Metallic`. Watch the red plastic transform into red chrome.

VISUALS: [Split-screen render of the preview sphere: left side with default Roughness (0.5), right side polished (0.07)].

### Level 3: Energy Overdrive (Emissive)

SOURCE CONCEPT: "Values set in the Details Panel or inline are automatically overidden if data is passed into the input pin associated with that property.".

THE RECIPE: Hold `M` and Left-Click for a `Multiply` node. Connect your red `Constant3Vector` to input `A`. Create a `Constant` ('1' + Click), set its value to `50.0`, and connect it to `B`. Connect the Multiply output to the `Emissive Color` input.

THE DIAGNOSIS: The material begins to glow intensely red, emitting a post-processing bloom effect around the sphere.

THE "ILLUMINATION": While physical properties like Roughness are strictly clamped between 0 and 1, emissive light operates in High Dynamic Range (HDR) and can exceed 1.0, instructing the GPU to emit physical light.

TINKER WITH IT: Change the multiplier to `1000.0` and watch the bloom completely flood the preview window.

VISUALS: [Graph highlighting the Multiply node scaling the color vector before reaching the Emissive Color pin].

### Level 4: The Texture Sample

SOURCE CONCEPT: "These calculations are done using data that is input to the Material from a variety of images (textures)...".

THE RECIPE: Hold `T` and Left-Click for a `TextureSample` node. In the Details panel, assign the default texture "T_Brick_Clay_New_D". Connect the top `RGB` pin directly to `Base Color`.

THE DIAGNOSIS: The sphere is no longer flat red; it is now photographically wrapped in a brick wall image.

THE "ILLUMINATION": A TextureSample node extracts color pixels and data from a 2D image file, executing a read from VRAM to override uniform constants.

TINKER WITH IT: Disconnect the `RGB` pin and connect only the `R` pin to Base Color. You will get a grayscale brick image based solely on the red channel data.

VISUALS: [Graph showing a Texture Sample node with a brick thumbnail replacing the Constant3Vector in the Base Color input].

### Level 5: Coordinate Math

SOURCE CONCEPT: "multiply my TexCoord by the Tiling Amount Parameter".

THE RECIPE: Hold `U` and Left-Click for a `TextureCoordinate` node. Generate a `Multiply` node. Connect the TexCoord to `A`. Create a `Constant` set to `3.0` and connect it to `B`. Connect the Multiply output to the `UVs` input of your TextureSample.

THE DIAGNOSIS: The brick texture shrinks, repeating exactly 3 times horizontally and vertically across the sphere's surface.

THE "ILLUMINATION": UVs are simply 2D mathematical coordinates (U and V, from 0 to 1). Multiplying them logically commands the sampling hardware to repeat the reading of the image array at a higher frequency.

TINKER WITH IT: Try adding (`Add` node) a constant to the TexCoord instead of multiplying. Notice how the texture physically slides (pans) across the mesh rather than scaling.

VISUALS: [Logical diagram of the flow: TexCoord -> Multiply (x3) -> TextureSample UVs].

### Level 6: The Parameter Bridge

SOURCE CONCEPT: "Master Materials... utilize customizable parameters... You can customize Material Instances without recompiling the parent Material.".

THE RECIPE: Right-click the `Constant` node controlling your Roughness (Level 2) and select `Convert to Parameter`. Name it `Roughness_Value`. Click `Save`. In the Content Browser, right-click your Material and select `Create Material Instance`. Double-click the new Instance to open it.

THE DIAGNOSIS: A lightweight window opens without the node graph. Check the box next to `Roughness_Value` and slide the number. The material updates in real time.

THE "ILLUMINATION": Converting a static node into a Parameter exposes a memory pointer to the CPU, allowing artists to adjust values on the fly without triggering a heavy HLSL shader recompile.

TINKER WITH IT: Change the parameter name in the Master Material to `Roughness Value` (with a space) and see how the Instance UI formats it cleanly.

VISUALS: [Screenshot of the Material Instance Editor showing the clean interface with the checkbox and slider of the exposed parameter].

### Level 7: Channel Packing (Component Mask)

SOURCE CONCEPT: "packed RGB texture with the following channel assignations: R: Ambient Occlusion, G: Roughness, B: Metallic".

THE RECIPE: In your Master Material, add a new TextureSample containing an ORM (Occlusion/Roughness/Metallic) texture. Pull a wire from its output and search for `ComponentMask`. In the Mask's Details panel, check *only* the `G` box. Connect this to `Roughness`.

THE DIAGNOSIS: The Material ignores the pink/yellow noise of the raw image and reads only the grayscale information from the Green channel to define which parts of the brick are shiny or rough.

| Channel | UE5 Standard Usage |
| ------ | ------ |
| **R** | Ambient Occlusion |
| **G** | Roughness |
| **B** | Metallic |

THE "ILLUMINATION": Storing three black-and-white masks in a single RGB image saves video memory and samplers; the Component Mask is a surgical tool to isolate the exact mathematical axis you need.

TINKER WITH IT: Check both the `R` and `G` boxes simultaneously in the mask and try to connect it to Roughness. The compiler will throw an error because Roughness strictly requires a 1D float, not a 2D vector.

VISUALS: [Node graph highlighting the Component Mask with the 'G' box checked in the Details panel].

### Level 8: Linear Interpolation (Lerp)

SOURCE CONCEPT: "Data is passed between Material Expressions by dragging a cable connection... combining Material expressions to achieve specific results".

THE RECIPE: Hold `L` and Left-Click for a `LinearInterpolate` (Lerp) node. Connect a Red Constant to `A` and a Blue Constant to `B`. Connect the `R` channel of your brick texture to the `Alpha` pin. Connect the Lerp output to `Base Color`.

THE DIAGNOSIS: The sphere is painted red and blue. The dark cracks of the brick mask render in Red, while the white faces of the bricks render in Blue.

THE "ILLUMINATION": The Lerp node is a hardware mixer (equivalent to `mix` in GLSL). Where the Alpha mask equals 0 (black), it outputs 100% of A. Where it equals 1 (white), it outputs 100% of B.

TINKER WITH IT: Swap the A and B connections and watch how the red and blue areas invert mathematically.

VISUALS: [Visual equation of Lerp: Input A (Red) + Input B (Blue) driven by Alpha (Brick Mask) = Final mixed texture].

### Level 9: Altering the Blend Mode

SOURCE CONCEPT: "Blend Mode – Defines how the Material blends with the pixels behind it. For example, Opaque... Translucent".

THE RECIPE: Click on the empty background of the Graph. In the Details panel, change `Blend Mode` to `Translucent`. Note that a new pin on the Main Node called `Opacity` becomes active. Connect a `Constant` of `0.5` to `Opacity`.

THE DIAGNOSIS: The sphere instantly becomes semi-transparent like colored glass, allowing you to see the background grid through it.

THE "ILLUMINATION": Changing the blend mode fundamentally alters the rendering state in HLSL, commanding the GPU to sort this object differently in the depth buffer, unlocking new inputs and disabling unused ones.

TINKER WITH IT: Connect a black-and-white mask texture to Opacity instead of a flat constant.

VISUALS: [Details panel showing the Blend Mode dropdown set to Translucent, with an arrow pointing to the newly enabled Opacity pin].

### Level 10: Swapping Shading Models

SOURCE CONCEPT: "Shading Model – Defines how the Material interacts with light... Clear Coat... enables the inputs for Clear Coat and Clear Coat Roughness.".

THE RECIPE: Revert the Blend Mode to `Opaque`. Change `Shading Model` to `Clear Coat`. Connect a Constant of `1.0` to the newly activated `Clear Coat` pin, and a Constant of `0.0` to `Clear Coat Roughness`.

THE DIAGNOSIS: The sphere appears to have a thick, perfectly smooth layer of wet lacquer applied over the base material, identical to automotive paint.

THE "ILLUMINATION": Shading Models swap out the entire physical BRDF (Bidirectional Reflectance Distribution Function) equation on the GPU, allowing specialized lighting algorithms for complex surfaces.

TINKER WITH IT: Change the Shading Model to `Unlit` and watch how all lighting, reflection, and shadow calculations disappear.

VISUALS: [Render of the sphere with the Clear Coat shading model active, showing a sharp secondary reflection layer over the base color].

### Level 11: The If Node and Divergence

SOURCE CONCEPT: "Does 'if' node really cost a bit more performance rather that 'custom' node... No.".

THE RECIPE: Search for an `If` node. Connect a TexCoord to `A` and a Constant of `0.5` to `B`. Connect a Red color to `A > B`, and a Green color to `A < B`. Connect the output to Base Color.

THE DIAGNOSIS: The sphere is split exactly in half diagonally: one side perfectly red and the other green, with a razor-sharp edge.

THE "ILLUMINATION": The `If` node performs a per-pixel mathematical branch, dynamically choosing an output based on a real-time comparison. While modern GPUs handle this well, excessive branching can cause thread divergence and stall parallel processing (warps).

TINKER WITH IT: Connect a different color to the `A == B` pin. Note how difficult it is to see the line because floating-point precision means numbers are rarely exactly equal.

VISUALS: [Graph of the IF node next to the half-red, half-green preview sphere].

### Level 12: Static Switches

SOURCE CONCEPT: "Static Switch Parameters can be used with Instanced Materials but... the override of a static switch will duplicate the material... leading to exponential growth.".

THE RECIPE: Search for `StaticSwitchParameter`. Name it `Use_Red`. Connect a Red color to `True`, a Blue color to `False`. Connect it to Base Color. Click Apply. Open your Material Instance and toggle the checkbox.

THE DIAGNOSIS: The material swaps instantly between Red and Blue when toggled in the Instance editor.

THE "ILLUMINATION": Unlike the `If` node, a Static Switch does not exist in the final GPU logic. It forces the Unreal compiler to generate two completely separate HLSL shader binaries and swaps between them before drawing the frame.

TINKER WITH IT: Add 5 more Static Switches to your graph. Understand that 2^5 = 32 different physical shader permutations will be generated and compiled.

VISUALS: [Material Instance UI showing the boolean checkbox for "Use_Red"].

### Level 13: Material Functions

SOURCE CONCEPT: "Material Functions enable you to package parts of a Material graph into a reusable asset... to streamline Material creation".

THE RECIPE: In the Content Browser, right-click -> Materials -> `Material Function`. Name it `MF_ColorTint`. Inside, add a `FunctionInput` (set Input Type to Vector3) and multiply it by a `Constant3Vector`. Connect this to the `FunctionOutput`. Save. Drag this function from the Content Browser directly into your Master Material.

THE DIAGNOSIS: Your multi-node mathematical operation is now collapsed into a single clean box with custom inputs and outputs.

THE "ILLUMINATION": Functions are the ultimate tool of the DRY (Don't Repeat Yourself) principle; they encapsulate logic, keeping Master Materials readable and ensuring updates propagate globally across a project.

TINKER WITH IT: Double-click the function node inside the Master Material. It will teleport you directly to its internal graph.

VISUALS: [The internal graph of a Material Function, ending in a purple Function Output node].

### Level 14: The Custom Node (HLSL)

SOURCE CONCEPT: "Material expressions that allow the use of custom, plain shader code... Custom nodes wrap your code inside CustomExpression#() functions".

THE RECIPE: Add a `Custom` node. In its Details panel, set the Code box to: `return (A > B) ? 1.0 : 0.0;`. Add two elements to the Inputs array and name them `A` and `B`. Connect textures or constants into them, and connect the output to Emissive.

THE DIAGNOSIS: The material performs the same split-screen logic as the visual `If` node, but it is written in raw HLSL text.

THE "ILLUMINATION": The Custom node allows you to bypass the visual graph for heavy mathematical operations (like iterative `for` loops) by directly injecting HLSL strings into the material compiler.

TINKER WITH IT: Type `return sin(A * B);` in the box and watch how the mathematical wave executes directly on the GPU.

VISUALS: [Details panel of the Custom node showing the Code text box and the named input array variables].

### Level 15: Exposing HLSL Internals

SOURCE CONCEPT: "See the generated code by looking in Window -> HLSL Code in the Material Editor.".

THE RECIPE: Go to the top menu bar of the Material Editor -> `Window` -> `HLSL Code`.

THE DIAGNOSIS: A massive text document opens, revealing thousands of lines of raw HLSL code.

THE "ILLUMINATION": The visual node graph is an abstraction. Unreal Engine uses a translator to convert your node connections into this monolithic text file that the DirectX/Vulkan compiler actually executes.

TINKER WITH IT: Click inside the text, press Ctrl+F, and search for `CustomExpression0` to find exactly where the engine copied and pasted the text from your Custom node.

VISUALS: [Screenshot of the HLSL Code window showing the dense shader text].

### Level 16: The Struct Hack

SOURCE CONCEPT: "struct definitions can be nested inside functions... you can start to build a library of custom functions".

THE RECIPE: In a Custom node, paste this exact code:

```hlsl
struct MyStruct {
    float3 GetColor() { return float3(0,1,0); }
};
MyStruct s;
return s.GetColor();

```

THE DIAGNOSIS: The material compiles successfully and turns green, executing a custom function method you defined yourself.

THE "ILLUMINATION": Because Unreal forcibly wraps the Custom node text inside an auto-generated function, standard global functions will fail to compile. Defining a `struct` tricks the HLSL compiler, allowing nested methods within the local scope.

TINKER WITH IT: Define a `for` loop inside the struct method to calculate complex iterative raymarching steps.

VISUALS: [Code block highlighting the `struct` wrapper trick inside the Custom node's text box].

### Level 17: External Code Inclusion

SOURCE CONCEPT: "Editing the shader file outside of UE4 is of course a much better dev experience... #include "/Project/myshader.ush"".

THE RECIPE: Assuming you mapped a virtual shader directory in your C++ project, set your Custom node code to:

```
#include "/Project/Shaders/MyShader.ush"
return 1;

```

THE DIAGNOSIS: The compiler searches the root folder of the project for a physical text file to inject directly into the shader during compilation.

THE "ILLUMINATION": You can completely bypass the primitive text box of the Details panel and write massive algorithms in a proper IDE like VSCode, letting Unreal retrieve and compile the file.

TINKER WITH IT: Intentionally misspell the file path and read the resulting red compilation error to understand how the parser fails.

VISUALS: [An external text editor (like VSCode) shown side-by-side with Unreal Engine's Material Editor].

### Level 18: Parameter Organization

SOURCE CONCEPT: "Parameters can and should be named... they can also be grouped together and ordered... Sort Priority lets you order your parameters".

THE RECIPE: Select your `Roughness_Value` parameter node. In the Details panel, type `01 - Surface` under `Group`. Change `Sort Priority` to `0`. Select another parameter, put it in `01 - Surface` too, but set `Sort Priority` to `1`.

THE DIAGNOSIS: Open your Material Instance. Your parameters are now neatly collapsed under a custom header folder, ordered exactly from top to bottom.

THE "ILLUMINATION": A Master Material is useless in production if environment artists cannot read it. Grouping and prioritizing is mandatory UX engineering to avoid overwhelming the team.

TINKER WITH IT: Leave a parameter without a group designation and watch it dump into the messy default "Global" category.

VISUALS: [Material Instance Details panel showing beautifully organized and sorted parameter groups].

### Level 19: Controlling Shader Explosion

SOURCE CONCEPT: "Complex materials can create a lot of shader permutations... memory footprint and load time... Overriding two will result in four materials and so on leading to exponential growth.".

THE RECIPE: Look at the `Usage` section in your Main Material Node's Details panel. By default, checkboxes like `Used with Skeletal Mesh` or `Used with Static Lighting` might be checked.

THE DIAGNOSIS: Every checked usage flag, mathematically multiplied by every Static Switch combination, generates another hidden shader variant taking up disk space and compile time.

THE "ILLUMINATION": True mastery is not adding features; it is strictly limiting the bounds of the Master Material to prevent crushing your game's memory budget and iteration speed.

TINKER WITH IT: Uncheck all usage flags. Try applying the material to a Skeletal Mesh character and watch the engine force a recompile to auto-check the required flag.

VISUALS: [Details panel focusing heavily on the Usage flag checkboxes].

### Level 20: Vertex Displacement (Customized UVs)

SOURCE CONCEPT: "Customized UVs: Feature that allows running calculations in the vertex shader to increase performance over running them per-pixel.".

THE RECIPE: In the Details panel of the Main Material Node, search for "Num Customized UVs" and set its value to `1`. Connect a `Time` node multiplied by a `Constant` to the new `Customized UV0` input pin. Then, connect a pure `TextureCoordinate` node (set to Index 0) to the `UVs` pins of your textures in the pixel shader.

THE DIAGNOSIS: The texture will scroll and animate exactly as if you had done the math directly connected to the texture, but the instruction count in the Stats window will drop drastically in the Base Pass shader section.

THE "ILLUMINATION": The Customized UVs node hijacks hardware interpolators to force the engine to calculate complex math (like scaling or panning textures) once per mesh vertex, instead of recalculating the same equation millions of times for every pixel drawn on the screen.

TINKER WITH IT: Use a non-linear math node (like noise or WorldPosition) inside Customized UV0 on a very low-resolution mesh and notice how the texture breaks visually due to the lack of vertex density to interpolate the curve.

VISUALS: [Screenshot of the Main Material Node showing the enabled Customized UV0 pin with connected time nodes, visually compared to the 'Stats' panel showing an instruction count drop].

***

**Navigation Index**

- Level 1: The Main Material Node
- Level 2: Physical Constraints
- Level 3: Energy Overdrive (Emissive)
- Level 4: The Texture Sample
- Level 5: Coordinate Math
- Level 6: The Parameter Bridge
- Level 7: Channel Packing (Component Mask)
- Level 8: Linear Interpolation (Lerp)
- Level 9: Altering the Blend Mode
- Level 10: Swapping Shading Models
- Level 11: The If Node and Divergence
- Level 12: Static Switches
- Level 13: Material Functions
- Level 14: The Custom Node (HLSL)
- Level 15: Exposing HLSL Internals
- Level 16: The Struct Hack
- Level 17: External Code Inclusion
- Level 18: Parameter Organization
- Level 19: Controlling Shader Explosion
- Level 20: Vertex Displacement (Customized UVs)
