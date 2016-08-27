# arcball_camera

Single-header single-function C/C++ immediate-mode camera for your graphics demos

Just call `arcball_camera_update` once per frame.

# OpenGL Example

![openglexample](http://i.imgur.com/DOX9gBX.gif)

Below is a fully-functional example program that works using OpenGL.

Creaet a new Visual Studio project, drop this file in it, and it should just work.

```
#define ARCBALL_CAMERA_IMPLEMENTATION
#include "arcball_camera.h"

#include <Windows.h>
#include <GL/gl.h>
#include <GL/glu.h>
#include <stdio.h>

#pragma comment(lib, "OpenGL32.lib")
#pragma comment(lib, "glu32.lib")

int g_mouse_delta;

LRESULT CALLBACK MyWndProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam)
{
    switch (message)
    {
    case WM_CLOSE:
        ExitProcess(0);
    case WM_MOUSEWHEEL:
        g_mouse_delta += GET_WHEEL_DELTA_WPARAM(wParam) / WHEEL_DELTA;
        break;
    }
    
    return DefWindowProc(hWnd, message, wParam, lParam);
}

int CALLBACK WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow)
{
    WNDCLASSEX wc;
    ZeroMemory(&wc, sizeof(wc));
    wc.cbSize = sizeof(wc);
    wc.style = CS_OWNDC;
    wc.lpfnWndProc = MyWndProc;
    wc.hInstance = hInstance;
    wc.hCursor = LoadCursor(NULL, IDC_ARROW);
    wc.hbrBackground = (HBRUSH)COLOR_BACKGROUND;
    wc.lpszClassName = TEXT("WindowClass");
    RegisterClassEx(&wc);

    RECT wr = { 0, 0, 640, 480 };
    AdjustWindowRect(&wr, 0, FALSE);
    HWND hWnd = CreateWindowEx(
        0, TEXT("WindowClass"),
        TEXT("BasicGL"),
        WS_OVERLAPPEDWINDOW,
        0, 0, wr.right - wr.left, wr.bottom - wr.top,
        0, 0, hInstance, 0);

    PIXELFORMATDESCRIPTOR pfd;
    ZeroMemory(&pfd, sizeof(pfd));
    pfd.nSize = sizeof(pfd);
    pfd.nVersion = 1;
    pfd.dwFlags = PFD_DRAW_TO_WINDOW | PFD_SUPPORT_OPENGL | PFD_DOUBLEBUFFER;
    pfd.iPixelType = PFD_TYPE_RGBA;
    pfd.cColorBits = 32;
    pfd.cDepthBits = 24;
    pfd.cStencilBits = 8;
    pfd.iLayerType = PFD_MAIN_PLANE;

    HDC hDC = GetDC(hWnd);

    int chosenPixelFormat = ChoosePixelFormat(hDC, &pfd);
    SetPixelFormat(hDC, chosenPixelFormat, &pfd);

    HGLRC hGLRC = wglCreateContext(hDC);
    wglMakeCurrent(hDC, hGLRC);

    ShowWindow(hWnd, SW_SHOWNORMAL);

    float pos[3] = { 3.0f, 3.0f, 3.0f };
    float target[3] = { 0.0f, 0.0f, 0.0f };

    // initialize "up" to be tangent to the sphere!
    // up = cross(cross(look, world_up), look)
    float up[3];
    {
        float look[3] = { target[0] - pos[0], target[1] - pos[1], target[2] - pos[2] };
        float look_len = sqrtf(look[0] * look[0] + look[1] * look[1] + look[2] * look[2]);
        look[0] /= look_len;
        look[1] /= look_len;
        look[2] /= look_len;

        float world_up[3] = { 0.0f, 1.0f, 0.0f };

        float across[3] = {
            look[1] * world_up[2] - look[2] * world_up[1],
            look[2] * world_up[0] - look[0] * world_up[2],
            look[0] * world_up[1] - look[1] * world_up[0],
        };

        up[0] = across[1] * look[2] - across[2] * look[1];
        up[1] = across[2] * look[0] - across[0] * look[2];
        up[2] = across[0] * look[1] - across[1] * look[0];
        
        float up_len = sqrtf(up[0] * up[0] + up[1] * up[1] + up[2] * up[2]);
        up[0] /= up_len;
        up[1] /= up_len;
        up[2] /= up_len;
    }

    LARGE_INTEGER then, now, freq;
    QueryPerformanceFrequency(&freq);
    QueryPerformanceCounter(&then);

    POINT oldcursor;
    GetCursorPos(&oldcursor);

    float oldview[16];

    while (!GetAsyncKeyState(VK_ESCAPE))
    {
        MSG msg;
        while (PeekMessage(&msg, NULL, 0, 0, PM_REMOVE))
        {
            TranslateMessage(&msg);
            DispatchMessage(&msg);
        }

        QueryPerformanceCounter(&now);
        float delta_time_sec = (float)(now.QuadPart - then.QuadPart) / freq.QuadPart;

        POINT cursor;
        GetCursorPos(&cursor);
        ScreenToClient(hWnd, &cursor);

        float view[16];
        arcball_camera_update(
            pos, target, up, view,
            delta_time_sec,
            0.1f, // zoom per tick
            0.5f, // pan speed
            3.0f, // rotation multiplier
            640, 480, // screen (window) size
            oldcursor.x, cursor.x,
            oldcursor.y, cursor.y,
            GetAsyncKeyState(VK_MBUTTON),
            GetAsyncKeyState(VK_RBUTTON),
            g_mouse_delta,
            0);

        if (memcmp(oldview, view, sizeof(float) * 16) != 0)
        {
            printf("\n");
            printf("pos: %f, %f, %f\n", pos[0], pos[1], pos[2]);
            printf("target: %f, %f, %f\n", target[0], target[1], target[2]);
            printf("view: %f %f %f %f\n"
                   "      %f %f %f %f\n"
                   "      %f %f %f %f\n"
                   "      %f %f %f %f\n",
                 view[0],  view[1],  view[2],  view[3],
                 view[4],  view[5],  view[6],  view[7],
                 view[8],  view[9], view[10], view[11],
                view[12], view[13], view[14], view[15]);
        }

        memcpy(oldview, view, sizeof(float) * 16);
        glClear(GL_COLOR_BUFFER_BIT);

        glMatrixMode(GL_PROJECTION);
        glLoadIdentity();
        gluPerspective(70.0, (double)(wr.right - wr.left) / (wr.bottom - wr.top), 0.001, 100.0);

        glMatrixMode(GL_MODELVIEW);
        glLoadMatrixf(view);

        glBegin(GL_TRIANGLES);
        glColor3ub(255, 0, 0);
        glVertex3f(-1.0f, -1.0f, 0.0f);
        glColor3ub(0, 255, 0);
        glVertex3f(1.0f, -1.0f, 0.0f);
        glColor3ub(0, 0, 255);
        glVertex3f(0.0f, 1.0f, 0.0f);
        glEnd();

        SwapBuffers(hDC);

        then = now;
        oldcursor = cursor;

        g_mouse_delta = 0;
    }

    return 0;
}
```
