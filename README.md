import bpy
import math
import random

# Clear existing scene
bpy.ops.object.select_all(action='SELECT')
bpy.ops.object.delete(use_global=False)

# Set render engine to Cycles for better quality
bpy.context.scene.render.engine = 'CYCLES'
bpy.context.scene.cycles.samples = 128
bpy.context.scene.render.resolution_x = 1920
bpy.context.scene.render.resolution_y = 1080

# Create camera
bpy.ops.object.camera_add(location=(0, -15, 3))
camera = bpy.context.object
camera.rotation_euler = (math.radians(85), 0, 0)
bpy.context.scene.camera = camera

# Create man figure (simple humanoid shape)
def create_man():
    # Body
    bpy.ops.mesh.primitive_cube_add(size=1, location=(0, 0, 1.5))
    body = bpy.context.object
    body.scale = (0.6, 0.3, 1)
    body.name = "Body"
    
    # Head
    bpy.ops.mesh.primitive_uv_sphere_add(radius=0.3, location=(0, 0, 2.8))
    head = bpy.context.object
    head.name = "Head"
    
    # Left Arm
    bpy.ops.mesh.primitive_cube_add(size=0.5, location=(-0.75, 0, 1.5))
    left_arm = bpy.context.object
    left_arm.scale = (0.3, 0.2, 1)
    left_arm.name = "LeftArm"
    
    # Right Arm
    bpy.ops.mesh.primitive_cube_add(size=0.5, location=(0.75, 0, 1.5))
    right_arm = bpy.context.object
    right_arm.scale = (0.3, 0.2, 1)
    right_arm.name = "RightArm"
    
    # Left Leg
    bpy.ops.mesh.primitive_cube_add(size=0.5, location=(-0.25, 0, 0.5))
    left_leg = bpy.context.object
    left_leg.scale = (0.25, 0.25, 1)
    left_leg.name = "LeftLeg"
    
    # Right Leg
    bpy.ops.mesh.primitive_cube_add(size=0.5, location=(0.25, 0, 0.5))
    right_leg = bpy.context.object
    right_leg.scale = (0.25, 0.25, 1)
    right_leg.name = "RightLeg"
    
    # Create dark clothing material
    mat = bpy.data.materials.new(name="ClothingMaterial")
    mat.use_nodes = True
    nodes = mat.node_tree.nodes
    nodes.clear()
    
    output = nodes.new(type='ShaderNodeOutputMaterial')
    bsdf = nodes.new(type='ShaderNodeBsdfPrincipled')
    bsdf.inputs['Base Color'].default_value = (0.1, 0.1, 0.15, 1)
    bsdf.inputs['Roughness'].default_value = 0.8
    
    mat.node_tree.links.new(bsdf.outputs['BSDF'], output.inputs['Surface'])
    
    # Apply material to all parts
    for obj in [body, head, left_arm, right_arm, left_leg, right_leg]:
        if obj.data.materials:
            obj.data.materials[0] = mat
        else:
            obj.data.materials.append(mat)
    
    # Join all parts
    bpy.ops.object.select_all(action='DESELECT')
    for obj in [body, head, left_arm, right_arm, left_leg, right_leg]:
        obj.select_set(True)
    bpy.context.view_layer.objects.active = body
    bpy.ops.object.join()
    body.name = "Man"
    
    return body

man = create_man()

# Create mountain using displacement modifier
def create_mountain(location, scale):
    bpy.ops.mesh.primitive_plane_add(size=20, location=location)
    mountain = bpy.context.object
    mountain.scale = scale
    
    # Subdivide for detail
    bpy.ops.object.mode_set(mode='EDIT')
    bpy.ops.mesh.subdivide(number_cuts=50)
    bpy.ops.object.mode_set(mode='OBJECT')
    
    # Add displacement modifier
    disp_mod = mountain.modifiers.new(name="Displace", type='DISPLACE')
    
    # Create texture for displacement
    tex = bpy.data.textures.new(name="MountainTexture", type='VORONOI')
    tex.noise_scale = 0.5
    disp_mod.texture = tex
    disp_mod.strength = 8
    disp_mod.mid_level = 0.3
    
    # Create snow material
    mat = bpy.data.materials.new(name="SnowMaterial")
    mat.use_nodes = True
    nodes = mat.node_tree.nodes
    nodes.clear()
    
    output = nodes.new(type='ShaderNodeOutputMaterial')
    bsdf = nodes.new(type='ShaderNodeBsdfPrincipled')
    bsdf.inputs['Base Color'].default_value = (0.95, 0.95, 1.0, 1)
    bsdf.inputs['Roughness'].default_value = 0.4
    bsdf.inputs['Specular IOR Level'].default_value = 0.5
    
    mat.node_tree.links.new(bsdf.outputs['BSDF'], output.inputs['Surface'])
    mountain.data.materials.append(mat)
    
    return mountain

# Create multiple mountains for depth
mountain1 = create_mountain((0, 20, 0), (2, 2, 1))
mountain2 = create_mountain((-15, 35, 0), (1.5, 1.5, 0.8))
mountain3 = create_mountain((20, 40, 0), (1.8, 1.8, 0.9))

# Create ground plane
bpy.ops.mesh.primitive_plane_add(size=50, location=(0, 0, 0))
ground = bpy.context.object
mat = bpy.data.materials.new(name="GroundSnow")
mat.use_nodes = True
nodes = mat.node_tree.nodes
bsdf = nodes.get("Principled BSDF")
bsdf.inputs['Base Color'].default_value = (0.9, 0.92, 0.95, 1)
bsdf.inputs['Roughness'].default_value = 0.7
ground.data.materials.append(mat)

# Create volumetric snow atmosphere
world = bpy.context.scene.world
world.use_nodes = True
nodes = world.node_tree.nodes
nodes.clear()

output = nodes.new(type='ShaderNodeOutputWorld')
background = nodes.new(type='ShaderNodeBackground')
background.inputs['Color'].default_value = (0.7, 0.75, 0.8, 1)
background.inputs['Strength'].default_value = 0.6

volume_scatter = nodes.new(type='ShaderNodeVolumeScatter')
volume_scatter.inputs['Density'].default_value = 0.05

world.node_tree.links.new(background.outputs['Background'], output.inputs['Surface'])
world.node_tree.links.new(volume_scatter.outputs['Volume'], output.inputs['Volume'])

# Add lighting
bpy.ops.object.light_add(type='SUN', location=(10, -10, 20))
sun = bpy.context.object
sun.data.energy = 2
sun.data.angle = math.radians(10)
sun.rotation_euler = (math.radians(45), 0, math.radians(30))

# Create particle system for snowfall
bpy.ops.mesh.primitive_plane_add(size=30, location=(0, 5, 15))
emitter = bpy.context.object
emitter.name = "SnowEmitter"

# Add particle system
particle_settings = bpy.data.particles.new(name="Snowfall")
particle_modifier = emitter.modifiers.new(name="Snowfall", type='PARTICLE_SYSTEM')
particle_system = emitter.particle_systems[particle_modifier.name]
particle_system.settings = particle_settings

# Configure particle settings for heavy snowfall
ps = particle_system.settings
ps.count = 5000
ps.lifetime = 120
ps.frame_start = 1
ps.frame_end = 250
ps.emit_from = 'FACE'
ps.normal_factor = -0.5
ps.factor_random = 0.1

# Physics
ps.physics_type = 'NEWTON'
ps.mass = 0.1
ps.use_multiply_size_mass = True
ps.particle_size = 0.02
ps.size_random = 0.5

# Render settings
ps.render_type = 'OBJECT'
bpy.ops.mesh.primitive_ico_sphere_add(subdivisions=1, radius=0.02)
snow_particle = bpy.context.object
snow_particle.name = "SnowParticle"
ps.instance_object = snow_particle
snow_particle.hide_render = False
snow_particle.hide_viewport = False

# Add snow material to particles
snow_mat = bpy.data.materials.new(name="SnowParticleMaterial")
snow_mat.use_nodes = True
nodes = snow_mat.node_tree.nodes
bsdf = nodes.get("Principled BSDF")
bsdf.inputs['Base Color'].default_value = (1, 1, 1, 1)
bsdf.inputs['Roughness'].default_value = 0.3
bsdf.inputs['Emission Color'].default_value = (1, 1, 1, 1)
bsdf.inputs['Emission Strength'].default_value = 0.1
snow_particle.data.materials.append(snow_mat)

# Gravity and wind
ps.effector_weights.gravity = 0.3
bpy.ops.object.effector_add(type='WIND', location=(0, -5, 10))
wind = bpy.context.object
wind.field.strength = 2
wind.field.noise = 0.5
wind.field.seed = random.randint(1, 1000)

# Setup camera animation (zoom in)
camera.location = (0, -25, 8)
camera.rotation_euler = (math.radians(75), 0, 0)
camera.keyframe_insert(data_path="location", frame=1)
camera.keyframe_insert(data_path="rotation_euler", frame=1)

# End position (zoomed in behind man)
camera.location = (0, -8, 4)
camera.rotation_euler = (math.radians(82), 0, 0)
camera.keyframe_insert(data_path="location", frame=150)
camera.keyframe_insert(data_path="rotation_euler", frame=150)

# Set animation length
bpy.context.scene.frame_start = 1
bpy.context.scene.frame_end = 150

# Add motion blur for cinematic effect
bpy.context.scene.render.use_motion_blur = True
bpy.context.scene.render.motion_blur_shutter = 0.5

# Add depth of field
camera.data.dof.use_dof = True
camera.data.dof.focus_distance = 15
camera.data.dof.aperture_fstop = 2.8

print("✓ Cinematic snowy mountain scene created successfully!")
print("✓ Camera zoom animation set from frame 1 to 150")
print("✓ Heavy snowfall particle system added")
print("✓ Press Spacebar to play animation")
print("✓ Go to Render > Render Animation to export")
