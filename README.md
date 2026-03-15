# Cs-2-Extarnel-Esp-Open-Source-
It only contains a health bar and a skeleton. If there's interest, I'll develop it further and make a GUI bypass version. I don't think you'll get banned in this version. It could be a basic tool for those who want to make Python ESPs; you can add it to your own cheats.
import pymem
import pymem.process
import time
import requests
from pynput import keyboard
import glfw
import OpenGL.GL as gl
import imgui
from imgui.integrations.glfw import GlfwRenderer
import win32gui
import win32con

WINDOW_WIDTH = 1920
WINDOW_HEIGHT = 1080
esp_active = True

# Offsetler 
offsets = requests.get('https://raw.githubusercontent.com/a2x/cs2-dumper/main/output/offsets.json').json()
client_dll = requests.get('https://raw.githubusercontent.com/a2x/cs2-dumper/main/output/client_dll.json').json()

client_classes = client_dll['client.dll']['classes']
dwEntityList = offsets['client.dll']['dwEntityList']
dwLocalPlayerPawn = offsets['client.dll']['dwLocalPlayerPawn']
dwViewMatrix = offsets['client.dll']['dwViewMatrix']
m_iTeamNum = client_classes['C_BaseEntity']['fields']['m_iTeamNum']
m_iHealth = client_classes['C_BaseEntity']['fields']['m_iHealth']
m_lifeState = client_classes['C_BaseEntity']['fields']['m_lifeState']
m_pGameSceneNode = client_classes['C_BaseEntity']['fields']['m_pGameSceneNode']
m_modelState = client_classes['CSkeletonInstance']['fields']['m_modelState']
m_hPlayerPawn = client_classes['CCSPlayerController']['fields']['m_hPlayerPawn']

pm = None
client = None

def w2s(x, y, z, vm):
    clip = vm[12]*x + vm[13]*y + vm[14]*z + vm[15]
    if clip < 0.01: return None
    nx = (vm[0]*x + vm[1]*y + vm[2]*z + vm[3]) / clip
    ny = (vm[4]*x + vm[5]*y + vm[6]*z + vm[7]) / clip
    if abs(nx) > 35 or abs(ny) > 35: return None
    sx = (WINDOW_WIDTH / 2) * (1 + nx)
    sy = (WINDOW_HEIGHT / 2) * (1 - ny)
    return (sx, sy)

def get_bone(bone_array, id):
    try:
        b = bone_array + id * 0x20
        return (pm.read_float(b), pm.read_float(b+4), pm.read_float(b+8))
    except:
        return None

def get_pawn(idx, elist):
    try:
        le = pm.read_longlong(elist + 0x8 * ((idx & 0x7FFF) >> 9) + 0x10)
        ctrl = pm.read_longlong(le + 0x70 * (idx & 0x1FF))
        h = pm.read_uint(ctrl + m_hPlayerPawn)
        if h in (0, 0xFFFFFFFF): return 0
        pe = pm.read_longlong(elist + 0x8 * ((h & 0x7FFF) >> 9) + 0x10)
        return pm.read_longlong(pe + 0x70 * (h & 0x1FF))
    except:
        return 0

def draw_skel(dl, bone_array, color, vm):
    bones = {"head":6,"neck":5,"pelvis":0,"lsh":9,"lel":10,"lha":11,"rsh":13,"rel":14,"rha":15,"lhi":22,"lkn":23,"lan":24,"rhi":25,"rkn":26,"ran":27}
    pos = {}
    for n, i in bones.items():
        p = get_bone(bone_array, i)
        if p: 
            s = w2s(*p, vm)
            if s: pos[n] = s
    if "head" not in pos or "pelvis" not in pos: return

    oc = imgui.get_color_u32_rgba(0,0,0,0.9)
    t = 1.4
    ot = t + 1.3

    def ln(a,b):
        if a in pos and b in pos:
            dl.add_line(*pos[a], *pos[b], oc, ot)
            dl.add_line(*pos[a], *pos[b], color, t)

    ln("head","neck"); ln("neck","pelvis")
    ln("neck","lsh"); ln("lsh","lel"); ln("lel","lha")
    ln("neck","rsh"); ln("rsh","rel"); ln("rel","rha")
    ln("pelvis","lhi"); ln("lhi","lkn"); ln("lkn","lan")
    ln("pelvis","rhi"); ln("rhi","rkn"); ln("rkn","ran")

def draw_esp(dl, vm, local_pawn, local_team, elist):
    if not esp_active: return
    for i in range(1,65):
        pawn = get_pawn(i, elist)
        if not pawn or pawn == local_pawn: continue
        try:
            team = pm.read_int(pawn + m_iTeamNum)
            if team == local_team or team not in (2,3): continue
            hp = pm.read_int(pawn + m_iHealth)
            if not 1 <= hp <= 100: continue
            if pm.read_uchar(pawn + m_lifeState) != 0: continue

            sn = pm.read_longlong(pawn + m_pGameSceneNode)
            if not sn: continue

            bone_array = None
            for off in (0x210, m_modelState+0x80, 0x1F0, 0x200):
                try:
                    val = pm.read_longlong(sn + off)
                    if val: bone_array = val; break
                except: pass
            if not bone_array: continue

            col = imgui.get_color_u32_rgba(1,0.18,0.18,0.98) if team==2 else imgui.get_color_u32_rgba(0.18,0.5,1,0.98)
            draw_skel(dl, bone_array, col, vm)

            # Health bar
            h3d = get_bone(bone_array, 6)
            if not h3d: continue
            h3d = (h3d[0], h3d[1], h3d[2]+3.5)
            hp_pos = w2s(*h3d, vm)
            if not hp_pos: continue

            by = hp_pos[1] - 28
            bw, bh = 36, 4
            bx1 = hp_pos[0] - bw/2
            bx2 = bx1 + bw
            ratio = hp / 100
            hpcol = imgui.get_color_u32_rgba(0,1,0,1) if hp>65 else imgui.get_color_u32_rgba(1,0.85,0,1) if hp>35 else imgui.get_color_u32_rgba(1,0.18,0.18,1)

            dl.add_rect_filled(bx1-2, by-bh/2-2, bx2+2, by+bh/2+2, imgui.get_color_u32_rgba(0,0,0,0.88))
            dl.add_rect_filled(bx1, by-bh/2, bx1+bw*ratio, by+bh/2, hpcol)
            dl.add_rect(bx1-2, by-bh/2-2, bx2+2, by+bh/2+2, imgui.get_color_u32_rgba(1,1,1,0.3), thickness=1)
        except:
            continue

def toggle(key):
    global esp_active
    if key == keyboard.Key.f1:
        esp_active = not esp_active

def main():
    global pm, client
    while True:
        try:
            pm = pymem.Pymem("cs2.exe")
            client = pymem.process.module_from_name(pm.process_handle, "client.dll").lpBaseOfDll
            break
        except:
            time.sleep(1)

    listener = keyboard.Listener(on_press=toggle)
    listener.start()

    glfw.init()
    glfw.window_hint(glfw.TRANSPARENT_FRAMEBUFFER, True)
    win = glfw.create_window(WINDOW_WIDTH, WINDOW_HEIGHT, " ", None, None)
    hwnd = glfw.get_win32_window(win)
    style = win32gui.GetWindowLong(hwnd, win32con.GWL_STYLE)
    win32gui.SetWindowLong(hwnd, win32con.GWL_STYLE, style & ~(win32con.WS_CAPTION | win32con.WS_THICKFRAME))
    ex = win32con.WS_EX_LAYERED | win32con.WS_EX_TRANSPARENT | win32con.WS_EX_TOPMOST
    win32gui.SetWindowLong(hwnd, win32con.GWL_EXSTYLE, ex)
    win32gui.SetWindowPos(hwnd, win32con.HWND_TOPMOST, -2, -2, 0, 0, win32con.SWP_NOSIZE)

    glfw.make_context_current(win)
    imgui.create_context()
    impl = GlfwRenderer(win)

    while not glfw.window_should_close(win):
        glfw.poll_events()
        impl.process_inputs()
        imgui.new_frame()
        imgui.set_next_window_size(WINDOW_WIDTH, WINDOW_HEIGHT)
        imgui.set_next_window_position(0, 0)
        imgui.begin("##", flags=imgui.WINDOW_NO_TITLE_BAR | imgui.WINDOW_NO_RESIZE | imgui.WINDOW_NO_SCROLLBAR | imgui.WINDOW_NO_COLLAPSE | imgui.WINDOW_NO_BACKGROUND)
        dl = imgui.get_window_draw_list()

        try:
            vm = [pm.read_float(client + dwViewMatrix + i*4) for i in range(16)]
            lp = pm.read_longlong(client + dwLocalPlayerPawn)
            if lp:
                lt = pm.read_int(lp + m_iTeamNum)
                el = pm.read_longlong(client + dwEntityList)
                draw_esp(dl, vm, lp, lt, el)
        except:
            pass

        imgui.end()
        gl.glClearColor(0,0,0,0)
        gl.glClear(gl.GL_COLOR_BUFFER_BIT)
        imgui.render()
        impl.render(imgui.get_draw_data())
        glfw.swap_buffers(win)
        time.sleep(0.001)

    impl.shutdown()
    glfw.terminate()
    listener.stop()

if __name__ == "__main__":
    main()

    print 
    
