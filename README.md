# Ray Tracing in One Weekend 번역
*Version 4.0.2, 2025-04-25 작업중*

<p align="center"><img src="https://raytracing.github.io/images/cover/CoverRTW1-small.jpg"></p>

>**이 글은 Peter Shirley의 [Ray Tracing in One Weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)를 번역한 것입니다.
Ray Tracing in One Weekend를 공부하면서 다시 한번 복습하는 느낌으로 번역을 해보려고 합니다. 영어가 서툴러 번역이 잘못되었을 수도 있으므로 잘못된 부분을 발견하신다면 지적해 주시면 감사하겠습니다.**

- ✅ **[1. Overview](./rtow_translation/01_overview.md)**
- ✅ **[2. Output an Image](./rtow_translation/02_output_an_image.md)**
    - [2.1 The PPM Image Format](./rtow_translation/02_output_an_image.md#21-the-ppm-image-format)
    - [2.2 Creating an Image File](./rtow_translation/02_output_an_image.md#22-creating-an-image-file)
    - [2.3 Adding a Progress Indicator](./rtow_translation/02_output_an_image.md#23-adding-a-progress-indicator)
- ✅ **[3. The vec3 Class](./rtow_translation/03_the_vec3_class.md)**
    - [3.1 Color Utility Functions](./rtow_translation/03_the_vec3_class.md#31-color-utility-functions)
- ✅ **[4. Rays, a Simple Camera, and Background](./rtow_translation/04_rays_a_simple_camera_and_background.md)**
    - [4.1 The ray Class](./rtow_translation/04_rays_a_simple_camera_and_background.md#41-the-ray-class)
    - [4.2 Sending Rays Into the Scene](./rtow_translation/04_rays_a_simple_camera_and_background.md#42-sending-rays-into-the-scene)
- ✅ **[5. Adding a Sphere](./rtow_translation/05_adding_a_sphere.md)** 
    - [5.1 Ray-Sphere Intersection](./rtow_translation/05_adding_a_sphere.md#51-ray-sphere-intersection)
    - [5.2 Creating Our First Raytraced Image](./rtow_translation/05_adding_a_sphere.md#52-creating-our-first-raytraced-image)
- ✅ **[6. Surface Normals and Multiple Objects](./rtow_translation/06_surface_normals_and_multiple_objects.md)**
    - [6.1 Shading with Surface Normals](./rtow_translation/06_surface_normals_and_multiple_objects.md#61-shading-with-surface-normals)
    - [6.2 Simplifying the Ray-Sphere Intersection Code](./rtow_translation/06_surface_normals_and_multiple_objects.md#62-simplifying-the-ray-sphere-intersection-code)
    - [6.3 An Abstraction for Hittable Objects](./rtow_translation/06_surface_normals_and_multiple_objects.md#63-an-abstraction-for-hittable-objects)
    - [6.4 Front Faces Versus Back Faces](./rtow_translation/06_surface_normals_and_multiple_objects.md#64-front-faces-versus-back-faces)
    - [6.5 A List of Hittable Objects](./rtow_translation/06_surface_normals_and_multiple_objects.md#65-a-list-of-hittable-objects)
    - [6.6 Some New C++ Features](./rtow_translation/06_surface_normals_and_multiple_objects.md#66-some-new-c-features)
    - [6.7 Common Constants and Utility Functions](./rtow_translation/06_surface_normals_and_multiple_objects.md#67-common-constants-and-utility-functions)
    - [6.8 An Interval Class](./rtow_translation/06_surface_normals_and_multiple_objects.md#68-an-interval-class)
- ✅ **[7. Moving Camera Code Into Own Class](./rtow_translation/07_moving_camera_code_into_own_class.md)**
- ✅ **[8. Antialiasing](./rtow_translation/08_antialiasing.md)**
    - [8.1 Some Random Number Utilities](./rtow_translation/08_antialiasing.md#81-some-random-number-utilities)
    - [8.2 Generating Pixels with Multiple Samples](./rtow_translation/08_antialiasing.md#82-generating-pixels-with-multiple-samples)
- **[9. Diffuse Materials](./rtow_translation/09_diffuse_materials.md)**
    - [9.1 A Simple Diffuse Material](./rtow_translation/09_diffuse_materials.md#91-a-simple-diffuse-material)
    - [9.2 Limiting the Number of Child Rays](./rtow_translation/09_diffuse_materials.md#92-limiting-the-number-of-child-rays)
    - [9.3 Using Gamma Correction for Accurate Color Intensity](./rtow_translation/09_diffuse_materials.md#93-using-gamma-correction-for-accurate-color-intensity)
    - [9.4 Fixing Shadow Acne](./rtow_translation/09_diffuse_materials.md#94-fixing-shadow-acne)
    - [9.5 True Lambertian Reflection](./rtow_translation/09_diffuse_materials.md#95-true-lambertian-reflection)
    - [9.6 An Alternative Diffuse Formulation](./rtow_translation/09_diffuse_materials.md#96-an-alternative-diffuse-formulation)
- **[10. Metal](./rtow_translation/10_metal.md)**
    - [10.1 An Abstract Class for Materials](./rtow_translation/10_metal.md#101-an-abstract-class-for-materials)
    - [10.2 A Data Structure to Describe Ray-Object Intersections](./rtow_translation/10_metal.md#102-a-data-structure-to-describe-ray-object-intersections)
    - [10.3 Modeling Light Scatter and Reflectance](./rtow_translation/10_metal.md#103-modeling-light-scatter-and-reflectance)
    - [10.4 Mirrored Light Reflection](./rtow_translation/10_metal.md#104-mirrored-light-reflection)
    - [10.5 A Scene with Metal Spheres](./rtow_translation/10_metal.md#105-a-scene-with-metal-spheres)
    - [10.6 Fuzzy Reflection](./rtow_translation/10_metal.md#106-fuzzy-reflection)
- **[11. Dielectrics](./rtow_translation/11_dielectrics.md)**
    - 11.1 Refraction
    - 11.2 Snell's Law
    - 11.3 Total Internal Reflection
    - 11.4 Schlick Approximation
    - 11.5 Modeling a Hollow Glass Sphere
- **[12. Positionable Camera](./rtow_translation/12_positionable_camera.md)**
    - [12.1 Camera Viewing Geometry](./rtow_translation/12_positionable_camera.md#121-camera-viewing-geometry)
    - [12.2 Positioning and Orienting the Camera](./rtow_translation/12_positionable_camera.md#122-positioning-and-orienting-the-camera)
- **[13. Defocus Blur](./rtow_translation/13_defocus_blur.md)**
    - 13.1 A Thin Lens Approximation
    - 13.2 Generating Sample Rays
- **[14. Where Next?](./rtow_translation/14_where_next.md)**
    - 14.1 A Final Render
    - 14.2 Next Steps
        - 14.2.1 Book 2: Ray Tracing: The Next Week
        - 14.2.2 Book 3: Ray Tracing: The Rest of Your Life
        - 14.2.3 Other Directions
