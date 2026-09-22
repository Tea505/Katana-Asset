import math
import bpy
import bmesh
from mathutils import Vector
 

BLADE_LENGTH   = 0.70
KISSAKI_LEN    = 0.040    
BASE_WIDTH     = 0.031    
TIP_WIDTH      = 0.022    
BASE_THICK     = 0.0072   
TIP_THICK      = 0.0048   
SORI           = -0.035   
TSUKA_LENGTH   = 0.250    
 
ITO_STRANDS    = 5        
ITO_TURNS      = 1.7      
 
SHOW_SAYA      = True    
CLEAR_SCENE    = True     
SETUP_SCENE    = True     
 
BODY_STEPS     = 140      
KISSAKI_STEPS  = 14

COL = None  
 
def lerp(a, b, t):
    return a + (b - a) * t
 
def clear_scene():
    for obj in list(bpy.data.objects):
        bpy.data.objects.remove(obj, do_unlink=True)
    for coll in list(bpy.data.collections):
        bpy.data.collections.remove(coll)
    for block in (bpy.data.meshes, bpy.data.curves, bpy.data.materials,
                  bpy.data.lights, bpy.data.cameras):
        for item in list(block):
            if item.users == 0:
                block.remove(item)
 
def make_material(name, color, metallic=0.0, roughness=0.5, bump=None):
    mat = bpy.data.materials.new(name)
    mat.use_nodes = True
    nt = mat.node_tree
    bsdf = nt.nodes.get("Principled BSDF")
    bsdf.inputs["Base Color"].default_value = (color[0], color[1], color[2], 1.0)
    bsdf.inputs["Metallic"].default_value = metallic
    bsdf.inputs["Roughness"].default_value = roughness
    if bump:
        coord = nt.nodes.new("ShaderNodeTexCoord")
        noise = nt.nodes.new("ShaderNodeTexNoise")
        noise.inputs["Scale"].default_value = bump[0]
        noise.inputs["Detail"].default_value = 6.0
        bmp = nt.nodes.new("ShaderNodeBump")
        bmp.inputs["Strength"].default_value = bump[1]
        bmp.inputs["Distance"].default_value = 0.002
        nt.links.new(coord.outputs["Object"], noise.inputs["Vector"])
        nt.links.new(noise.outputs["Fac"], bmp.inputs["Height"])
        nt.links.new(bmp.outputs["Normal"], bsdf.inputs["Normal"])
    return mat
 
def make_object(name, bm, mats, sharp_deg=35.0):
    bmesh.ops.recalc_face_normals(bm, faces=bm.faces[:])
    thr = math.radians(sharp_deg)
    for f in bm.faces:
        f.smooth = True
    for e in bm.edges:
        if len(e.link_faces) == 2 and e.calc_face_angle(0.0) > thr:
            e.smooth = False           
    me = bpy.data.meshes.new(name)
    bm.to_mesh(me)
    bm.free()
    if hasattr(me, "use_auto_smooth"):  
        me.use_auto_smooth = True
        me.auto_smooth_angle = math.radians(180.0)
    for m in mats:
        me.materials.append(m)
    obj = bpy.data.objects.new(name, me)
    COL.objects.link(obj)
    return obj
 
def se_point(a, ry, rz, n):
    c, s = math.cos(a), math.sin(a)
    e = 2.0 / n
    return (ry * math.copysign(abs(c) ** e, c),
            rz * math.copysign(abs(s) ** e, s))
 
def loft(name, stations, mat, segs=32, n=2.0, sharp=35.0, caps=(True, True)):
    bm = bmesh.new()
    rings = []
    for (x, zc, ry, rz) in stations:
        ring = []
        for k in range(segs):
            y, z = se_point(2.0 * math.pi * k / segs, ry, rz, n)
            ring.append(bm.verts.new((x, y, zc + z)))
        rings.append(ring)
    for i in range(len(rings) - 1):
        a, b = rings[i], rings[i + 1]
        for k in range(segs):
            k2 = (k + 1) % segs
            bm.faces.new((a[k], a[k2], b[k2], b[k]))
    if caps[0]:
        bm.faces.new(rings[0])
    if caps[1]:
        bm.faces.new(rings[-1])
    return make_object(name, bm, [mat], sharp)
 
def hamon_height(x):
    return (0.16
            + 0.045 * math.sin(x * 260.0)
            + 0.030 * math.sin(x * 570.0 + 1.3)
            + 0.015 * math.sin(x * 1300.0 + 0.4))
 
def build_blade(mat_steel, mat_hamon):
    L, K = BLADE_LENGTH, KISSAKI_LEN
    xk = L - K
 
    xs = [xk * i / BODY_STEPS for i in range(BODY_STEPS + 1)]
    xs += [xk + K * j / KISSAKI_STEPS for j in range(1, KISSAKI_STEPS + 1)]
 
    bm = bmesh.new()
    rings = []
    for x in xs:
        u = x / L
        w = lerp(BASE_WIDTH, TIP_WIDTH, u)
        t = lerp(BASE_THICK, TIP_THICK, u)
        zc = -SORI * u * u                       
        f = max(0.0, (x - xk) / K)               
 
        if f >= 0.9999:                          
            rings.append([bm.verts.new((x, 0.0, zc + w / 2.0))])
            continue
 
        g = f ** 2                               
        zE, zM = -w / 2.0, w / 2.0               
        zS = zE + 0.68 * w                      
        zH = zE + hamon_height(x) * w            
        th = t * (1.0 - f ** 3)                  
        yS = th / 2.0
        yH = yS * (zH - zE) / (zS - zE)          
        yM = 0.8 * yS                            
 
        prof = [(0, zE), (yH, zH), (yS, zS), (yM, zM), (-yM, zM), (-yS, zS), (-yH, zH)]
        ring = [bm.verts.new((x, y, zc + z + (zM - z) * g)) for (y, z) in prof]
        rings.append(ring)
 
    for i in range(len(rings) - 1):
        a, b = rings[i], rings[i + 1]
        for k in range(7):
            k2 = (k + 1) % 7
            if len(b) == 1:
                fc = bm.faces.new((a[k], a[k2], b[0]))
            else:
                fc = bm.faces.new((a[k], a[k2], b[k2], b[k]))
            fc.material_index = 1 if k in (0, 6) else 0
    bm.faces.new(rings[0][::-1])                
 
    return make_object("Blade", bm, [mat_steel, mat_hamon], sharp_deg=8.0)
 
def build_habaki(mat):
    st = [(0.000, 0, 0.0056, 0.0176),
          (0.003, 0, 0.0056, 0.0176),
          (0.028, 0, 0.0049, 0.0169),
          (0.031, 0, 0.0045, 0.0164)]
    return loft("Habaki", st, mat, segs=48, n=4.0)
 
def build_tsuba(mat_iron, mat_gold):
    r = 0.037
    tsuba = loft("Tsuba", [(-0.0015, 0, r - 0.002, r - 0.002),
                           (-0.0022, 0, r, r),
                           (-0.0078, 0, r, r),
                           (-0.0085, 0, r - 0.002, r - 0.002)],
                 mat_iron, segs=96)
    s1 = loft("Seppa_Front", [(0.0, 0, 0.0165, 0.0165), (-0.0015, 0, 0.0165, 0.0165)],
              mat_gold, segs=48)
    s2 = loft("Seppa_Back", [(-0.0085, 0, 0.0165, 0.0165), (-0.0100, 0, 0.0165, 0.0165)],
              mat_gold, segs=48)
    return tsuba, s1, s2

TSUKA_N   = 2.4                 
TSUKA_X0  = -0.010              
FIT       = 0.0034              

def core_end_x():
    return TSUKA_X0 - (TSUKA_LENGTH - 0.015)
 
def core_radii(x):
    s = (TSUKA_X0 - x) / (TSUKA_X0 - core_end_x())
    s = min(1.0, max(0.0, s))
    ry = 0.0128 - 0.0018 * s + 0.0009 * math.sin(math.pi * s)
    rz = 0.0165 - 0.0022 * s + 0.0012 * math.sin(math.pi * s)
    return ry, rz
 
def build_tsuka(mat_skin):
    xe = core_end_x()
    st = []
    for i in range(27):
        x = lerp(TSUKA_X0, xe, i / 26.0)
        ry, rz = core_radii(x)
        st.append((x, 0.0, ry, rz))
    return loft("Tsuka", st, mat_skin, segs=32, n=TSUKA_N)
 
def build_fuchi(mat):
    prof = [(-0.0100, 0.0026), (-0.0108, 0.0032), (-0.0300, FIT), (-0.0330, 0.0022)]
    st = []
    for x, off in prof:
        ry, rz = core_radii(x)
        st.append((x, 0.0, ry + off, rz + off))
    return loft("Fuchi", st, mat, segs=32, n=TSUKA_N)
 
def build_kashira(mat):
    x0 = -0.2300
    ry0, rz0 = core_radii(x0)
    st = []
    for j in range(9):
        th = math.radians(80.0 * j / 8.0)
        sc = math.cos(th) ** 0.55
        st.append((x0 - 0.030 * math.sin(th), 0.0,
                   (ry0 + FIT) * sc, (rz0 + FIT) * sc))
    return loft("Kashira", st, mat, segs=32, n=TSUKA_N)
 
def build_ito(mat):
    x_start, x_end = -0.028, -0.2325
    N, T = ITO_STRANDS, ITO_TURNS
    pts_per = 260
    delta = 1.0 / (2.0 * N * T)            
    A, BASE = 0.0006, 0.0004              
 
    cu = bpy.data.curves.new("Ito", 'CURVE')
    cu.dimensions = '3D'
    cu.bevel_depth = 0.0016
    cu.bevel_resolution = 2
    cu.use_fill_caps = True
 
    for direction in (+1, -1):
        for k in range(N):
            phase = 2.0 * math.pi * k / N
            sp = cu.splines.new('POLY')
            sp.points.add(pts_per - 1)
            for i in range(pts_per):
                s = i / (pts_per - 1)
                x = lerp(x_start, x_end, s)
                ang = phase + direction * 2.0 * math.pi * T * s
                off = BASE + direction * A * math.cos(math.pi * s / delta)
                ry, rz = core_radii(x)
                y, z = se_point(ang, ry + off, rz + off, TSUKA_N)
                sp.points[i].co = (x, y, z, 1.0)
    cu.materials.append(mat)
    obj = bpy.data.objects.new("Ito_Wrap", cu)
    COL.objects.link(obj)
    for p in obj.data.splines:
        p.use_smooth = True
    return obj
 
def build_saya(mat_lacquer, mat_horn):
    xe = BLADE_LENGTH + 0.004
    n_body = 60
    ry_a, rz_a = 0.0105, 0.0195
    ry_b, rz_b = 0.0072, 0.0135
    st = []
    for i in range(n_body + 1):
        x = xe * i / n_body
        u = x / BLADE_LENGTH
        st.append((x, -SORI * u * u, lerp(ry_a, ry_b, u), lerp(rz_a, rz_b, u)))
    ry_e, rz_e = st[-1][2], st[-1][3]
    for j in range(1, 9):                        
        th = math.radians(80.0 * j / 8.0)
        sc = math.cos(th) ** 0.55
        x = xe + 0.014 * math.sin(th)
        u = x / BLADE_LENGTH
        st.append((x, -SORI * u * u, ry_e * sc, rz_e * sc))
    saya = loft("Saya", st, mat_lacquer, segs=40, n=2.6)
 
    koi = loft("Koiguchi", [(0.0, 0, 0.0111, 0.0206), (0.018, 0, 0.0111, 0.0206)],
               mat_horn, segs=40, n=2.6)
    for o in (saya, koi):
        o.location = (0.0, 0.0, -0.085)
    return saya, koi
 
def add_area_light(name, loc, size, energy, target, color=(1, 1, 1)):
    ld = bpy.data.lights.new(name, 'AREA')
    ld.shape = 'RECTANGLE'
    ld.size, ld.size_y = size
    ld.energy = energy
    ld.color = color
    ob = bpy.data.objects.new(name, ld)
    COL.objects.link(ob)
    ob.location = loc
    ob.rotation_euler = (Vector(target) - Vector(loc)).to_track_quat('-Z', 'Y').to_euler()
    return ob
 
def setup_scene():
    scn = bpy.context.scene
    scn.render.engine = 'CYCLES'
    try:
        scn.cycles.samples = 128
    except Exception:
        pass
    scn.render.resolution_x, scn.render.resolution_y = 1920, 1080
 
    if scn.world is None:
        scn.world = bpy.data.worlds.new("World")
    scn.world.use_nodes = True
    bg = scn.world.node_tree.nodes.get("Background")
    if bg:
        bg.inputs["Color"].default_value = (0.015, 0.017, 0.022, 1.0)
        bg.inputs["Strength"].default_value = 1.0
 
    zc = -0.04 if SHOW_SAYA else 0.0
    target = (0.22, 0.0, zc)
 
    add_area_light("Key_Soft",   (0.2, -2.0, 0.9),  (1.8, 0.7), 500, target)
    add_area_light("Top_Strip",  (0.25, -0.5, 1.3), (1.6, 0.15), 350, target, (1.0, 0.97, 0.92))
    add_area_light("Rim_Cool",   (0.3, 1.5, 0.3),   (1.6, 0.3), 300, target, (0.7, 0.8, 1.0))
    add_area_light("Low_Strip",  (0.25, -1.0, -0.8), (1.6, 0.15), 200, target, (1.0, 0.9, 0.8))
 
    cam_data = bpy.data.cameras.new("Camera")
    cam_data.lens = 70
    cam = bpy.data.objects.new("Camera", cam_data)
    COL.objects.link(cam)
    cam.location = (0.05, -2.3, 0.30 + zc)
    cam.rotation_euler = (Vector(target) - Vector(cam.location)).to_track_quat('-Z', 'Y').to_euler()
    scn.camera = cam

def main():
    global COL
    if CLEAR_SCENE:
        clear_scene()
    COL = bpy.data.collections.new("Katana")
    bpy.context.scene.collection.children.link(COL)
 
    steel   = make_material("Steel",        (0.32, 0.34, 0.37), 1.0, 0.22)
    hamon   = make_material("Hamon",        (0.88, 0.90, 0.93), 1.0, 0.10)
    gold    = make_material("Gold",         (0.83, 0.62, 0.22), 1.0, 0.30)
    iron    = make_material("Tsuba_Iron",   (0.06, 0.06, 0.065), 1.0, 0.45, bump=(900, 0.15))
    shakudo = make_material("Shakudo",      (0.04, 0.035, 0.04), 1.0, 0.28)
    skin    = make_material("Same_Skin",    (0.80, 0.78, 0.70), 0.0, 0.55, bump=(1400, 0.35))
    ito     = make_material("Ito_Silk",     (0.025, 0.035, 0.09), 0.0, 0.55, bump=(1800, 0.25))
    lacquer = make_material("Lacquer_Black", (0.004, 0.004, 0.005), 0.0, 0.08)
    horn    = make_material("Horn",         (0.05, 0.035, 0.025), 0.0, 0.25)
 
    build_blade(steel, hamon)
    build_habaki(gold)
    build_tsuba(iron, gold)
    build_tsuka(skin)
    build_fuchi(shakudo)
    build_kashira(shakudo)
    build_ito(ito)
 
    if SHOW_SAYA:
        build_saya(lacquer, horn)
 
    if SETUP_SCENE:
        setup_scene()
 
main()
