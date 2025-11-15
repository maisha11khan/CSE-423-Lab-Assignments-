from OpenGL.GL import *
from OpenGL.GLUT import *
from OpenGL.GLU import *
import random
import math

WINDOW_WIDTH = 800
WINDOW_HEIGHT = 600
#set to none bc dont have any values assigned yet
rain_system = None
environment = None
house = None

class Environment:
    """Manages day/night cycle and background"""
    def __init__(self):
        self.time_of_day = 0.0 #initial value starts from night

    def transition_to_day(self):
        if self.time_of_day < 1.0:
            self.time_of_day += 0.05

    def transition_to_night(self):
        if self.time_of_day > 0.0:
            self.time_of_day -= 0.05
    #claculate gradual change from night to day
    def calculate_value(self, night_value, day_value):
        return night_value + (day_value - night_value) * self.time_of_day
    #changes color for smooth transition. adjusts color with transition
    def get_sky_color(self):
        r = self.calculate_value(0.05, 0.53)
        g = self.calculate_value(0.05, 0.81)
        b = self.calculate_value(0.15, 0.92)
        return (r, g, b)

    def get_ground_color(self):
        r = self.calculate_value(0.15, 0.45)
        g = self.calculate_value(0.25, 0.6)
        b = self.calculate_value(0.1, 0.25)
        return (r, g, b)

    def get_rain_color(self):
        r = self.calculate_value(0.6, 0.4)
        g = self.calculate_value(0.7, 0.5)
        b = self.calculate_value(0.8, 0.7)
        return (r, g, b)

class Rain:
    def __init__(self):
        self.reset() #raindrop properties(horizontal,vertical,size,speed)
        #random vertical position if RD as per screen height top to bottom
        self.y = random.uniform(-WINDOW_HEIGHT / 2, WINDOW_HEIGHT / 2)

    def reset(self):
        values = [0, 0, 0, 0]
        for i in range(4):
            #RD starts at a random horizontal range
            if i == 0:
                values[i] = random.uniform(-WINDOW_WIDTH / 2, WINDOW_WIDTH / 2)
            #starts at random vertical pos so that RD fall in the screen within window
            elif i == 1:
                values[i] = WINDOW_HEIGHT / 2 + random.uniform(0, 100)
            elif i == 2:#rain speed
                values[i] = random.uniform(4, 7)
            else:#rain len
                values[i] = random.uniform(15, 25)
        self.x, self.y, self.velocity, self.length = values[0], values[1], values[2], values[3]
    #update pos of RD and reset it
    def update(self):
        self.y = self.y - self.velocity
        if self.y < -WINDOW_HEIGHT / 2 - 50:
            self.reset()

class RainSystem:
    def __init__(self, num_drops=200):
        self.drops = []
        for i in range(num_drops):
            self.drops.append(Rain())
        self.wind_angle = 0.0

    def bend_left(self):
        new_val = self.wind_angle - 0.15
        if new_val < -2.5:
            new_val = -2.5
        self.wind_angle = new_val

    def bend_right(self):
        new_val = self.wind_angle + 0.15
        if new_val > 2.5: #doesn't exceed the maximum value
            new_val = 2.5
        self.wind_angle = new_val

    def update(self):
        for i in range(len(self.drops)):
            self.drops[i].update()

    def render(self, color):
        glColor3f(color[0], color[1], color[2])
        glLineWidth(1.5)
        for i in range(len(self.drops)):
            drop = self.drops[i]
            x_offset = self.wind_angle * 10
            glBegin(GL_LINES)
            glVertex2f(drop.x, drop.y)
            glVertex2f(drop.x + x_offset, drop.y - drop.length)
            glEnd()

class House:
    def __init__(self):
        self.pos = (0, -50)

    def render(self):
        px, py = self.pos
        part = 0
        while part < 7:
            if part == 0:
                self._draw_ground(py)
            elif part == 1:
                self._draw_trees(py)
            elif part == 2:
                self._draw_house_body(px, py)
            elif part == 3:
                self._draw_roof(px, py)
            elif part == 4:
                self._draw_chimney(px, py)
            elif part == 5:
                self._draw_door(px, py)
            elif part == 6:
                self._draw_windows(px, py)
            part = part + 1

    def _draw_ground(self, py):
        ground_y = py - 50 #position the ground below the house
        col = environment.get_ground_color()
        glColor3f(col[0], col[1], col[2])

        glBegin(GL_TRIANGLES)
        for part in range(2):
            #forms the upper part of the ground
            if part == 0:
                glVertex2f(-WINDOW_WIDTH/2, ground_y)
                glVertex2f(WINDOW_WIDTH/2, ground_y)
                glVertex2f(WINDOW_WIDTH/2, -WINDOW_HEIGHT/2)
            else: #bottom part of the ground
                glVertex2f(-WINDOW_WIDTH/2, ground_y)
                glVertex2f(WINDOW_WIDTH/2, -WINDOW_HEIGHT/2)
                glVertex2f(-WINDOW_WIDTH/2, -WINDOW_HEIGHT/2)
        glEnd()

    def _draw_trees(self, py): #multiple tree placement
        tree_positions = [-350, -280, -210, 220, 290, 360]
        for i in range(len(tree_positions)):
            tx = tree_positions[i]
            self._draw_single_tree(tx, py - 50)

    def _draw_single_tree(self, x, y): #how 1 tree should be drawn
        glColor3f(0.1, 0.5, 0.1)
        i = 0
        while i < 3:
            offset = i * 15
            glBegin(GL_TRIANGLES)
            glVertex2f(x - 20, y + offset)
            glVertex2f(x, y + 40 + offset)
            glVertex2f(x + 20, y + offset)
            glEnd()
            i = i + 1

        glColor3f(0.4, 0.25, 0.1)
        trunk_points = [
            (x - 5, y), (x + 5, y), (x + 5, y - 20),
            (x - 5, y), (x + 5, y - 20), (x - 5, y - 20)
        ]
        glBegin(GL_TRIANGLES)
        j = 0
        while j < len(trunk_points):
            glVertex2f(trunk_points[j][0], trunk_points[j][1])
            j = j + 1
        glEnd()

    def _draw_house_body(self, px, py):
        glColor3f(0.9, 0.85, 0.7)
        glBegin(GL_TRIANGLES)
        for part in range(2):
            if part == 0:
                glVertex2f(px - 180, py - 50)
                glVertex2f(px + 180, py - 50)
                glVertex2f(px + 180, py + 120)
            else:
                glVertex2f(px - 180, py - 50)
                glVertex2f(px + 180, py + 120)
                glVertex2f(px - 180, py + 120)
        glEnd()

    def _draw_roof(self, px, py):
        types = ["triangle", "lines"]
        for t in types:
            if t == "triangle":
                glColor3f(0.65, 0.3, 0.4)
                glBegin(GL_TRIANGLES)
                glVertex2f(px - 200, py + 120)
                glVertex2f(px, py + 230)
                glVertex2f(px + 200, py + 120)
                glEnd()
            else:
                glColor3f(0.5, 0.2, 0.3)
                glLineWidth(3)
                glBegin(GL_LINES)
                glVertex2f(px - 200, py + 120)
                glVertex2f(px, py + 230)
                glVertex2f(px, py + 230)
                glVertex2f(px + 200, py + 120)
                glEnd()

    def _draw_chimney(self, px, py):
        parts = ["body", "cap"]
        for part in parts:
            if part == "body":
                glColor3f(0.5, 0.3, 0.25)
                glBegin(GL_TRIANGLES)
                glVertex2f(px + 90, py + 150)
                glVertex2f(px + 130, py + 150)
                glVertex2f(px + 130, py + 210)
                glVertex2f(px + 90, py + 150)
                glVertex2f(px + 130, py + 210)
                glVertex2f(px + 90, py + 210)
                glEnd()
            else:
                glColor3f(0.4, 0.2, 0.15)
                glBegin(GL_TRIANGLES)
                glVertex2f(px + 80, py + 210)
                glVertex2f(px + 140, py + 210)
                glVertex2f(px + 140, py + 225)
                glVertex2f(px + 80, py + 210)
                glVertex2f(px + 140, py + 225)
                glVertex2f(px + 80, py + 225)
                glEnd()

    def _draw_door(self, px, py):
        glColor3f(0.35, 0.2, 0.15)
        glBegin(GL_TRIANGLES)
        coords = [
            (px - 35, py - 50), (px + 35, py - 50), (px + 35, py + 60),
            (px - 35, py - 50), (px + 35, py + 60), (px - 35, py + 60)
        ]
        i = 0
        while i < len(coords):
            glVertex2f(coords[i][0], coords[i][1])
            i = i + 1
        glEnd()

        glColor3f(0.25, 0.15, 0.1)
        glLineWidth(2)
        glBegin(GL_LINES)
        glVertex2f(px, py - 50)
        glVertex2f(px, py + 60)
        glVertex2f(px - 35, py + 5)
        glVertex2f(px + 35, py + 5)
        glEnd()

        glColor3f(0.85, 0.75, 0.3)
        glPointSize(6)
        glBegin(GL_POINTS)
        glVertex2f(px + 20, py + 5)
        glEnd()

    def _draw_windows(self, px, py):
        positions = [(-110, py + 35), (110, py + 35)]
        for wx, wy in positions:
            glColor3f(0.6, 0.85, 0.95)
            glBegin(GL_TRIANGLES)
            tri_points = [
                (wx - 35, wy - 30), (wx + 35, wy - 30), (wx + 35, wy + 30),
                (wx - 35, wy - 30), (wx + 35, wy + 30), (wx - 35, wy + 30)
            ]
            for point in tri_points:
                glVertex2f(point[0], point[1])
            glEnd()

            glColor3f(0.3, 0.2, 0.15)
            glLineWidth(3)
            lines = [
                (wx, wy - 30, wx, wy + 30),
                (wx - 35, wy, wx + 35, wy),
                (wx - 35, wy - 30, wx + 35, wy - 30),
                (wx + 35, wy - 30, wx + 35, wy + 30),
                (wx + 35, wy + 30, wx - 35, wy + 30),
                (wx - 35, wy + 30, wx - 35, wy - 30)
            ]
            glBegin(GL_LINES)
            for x1, y1, x2, y2 in lines:
                glVertex2f(x1, y1)
                glVertex2f(x2, y2)
            glEnd()

def initialize(): #setup openGl enviorment
    
    global rain_system, environment, house

    count = 0
    while count < 3:
        if count == 0:
            glViewport(0, 0, WINDOW_WIDTH, WINDOW_HEIGHT) #area of the window
        elif count == 1:
            glMatrixMode(GL_PROJECTION)# 3D objects are mapped to 2D
            glLoadIdentity()# project mat ---> identitry. clears previous transformation
            glOrtho(-WINDOW_WIDTH / 2, WINDOW_WIDTH / 2,
                    -WINDOW_HEIGHT / 2, WINDOW_HEIGHT / 2, -1, 1)#sets cord system for 2D scene
        else:
            glMatrixMode(GL_MODELVIEW)
            glLoadIdentity()
        count = count + 1
    #Initializing Objects
    environment = Environment()
    rain_system = RainSystem(200)
    house = House()

#draw everything that appears on screen
def render_scene():
    """Main display callback - draws background manually"""
    # Draw background sky manually using triangles
    sky_col = environment.get_sky_color()
    
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)#remove previous content.starts from fresh frame.
    glLoadIdentity()

    # Draw sky background as two large triangles.sky color adjusting with transition
    glColor3f(sky_col[0], sky_col[1], sky_col[2])
    glBegin(GL_TRIANGLES)
    glVertex2f(-WINDOW_WIDTH/2, -WINDOW_HEIGHT/2)
    glVertex2f(WINDOW_WIDTH/2, -WINDOW_HEIGHT/2)
    glVertex2f(WINDOW_WIDTH/2, WINDOW_HEIGHT/2)
    
    glVertex2f(-WINDOW_WIDTH/2, -WINDOW_HEIGHT/2)
    glVertex2f(WINDOW_WIDTH/2, WINDOW_HEIGHT/2)
    glVertex2f(-WINDOW_WIDTH/2, WINDOW_HEIGHT/2)
    glEnd()

    # Draw objects
    items = [house, rain_system]
    index = 0
    while index < len(items):
        if index == 0:
            items[index].render()#draws the object it is called on
        else:
            rain_color = environment.get_rain_color()
            items[index].render(rain_color)
        index = index + 1

    glutSwapBuffers()#makes the new frame visible on the screen.


def update_animation():
    
    do_update = True
    if do_update:
        rain_system.update()

    glutPostRedisplay()#re render so that RD's are seen


def handle_keyboard(key, x, y):
    """Keyboard handler"""
    day_keys = [b'd', b'D']
    night_keys = [b'n', b'N']

    if key in day_keys:
        environment.transition_to_day()
    elif key in night_keys:
        environment.transition_to_night()

    glutPostRedisplay()


def handle_special_keys(key, x, y):
    """Special keys handler"""
    left_keys = [GLUT_KEY_LEFT]
    right_keys = [GLUT_KEY_RIGHT]

    if key in left_keys:
        rain_system.bend_left()
    elif key in right_keys:
        rain_system.bend_right()

    glutPostRedisplay()


def main():
    """Main function"""
    glutInit()
    glutInitDisplayMode(GLUT_RGBA)
    glutInitWindowSize(WINDOW_WIDTH, WINDOW_HEIGHT)#sets size of window
    glutInitWindowPosition(100, 100)#sets window pos
    glutCreateWindow(b"Rainfall House Animation - OpenGL")

    initialize()

    # Register callbacks
    cb_list = ["display", "idle", "keyboard", "special"]
    for cb in cb_list:
        if cb == "display":
            glutDisplayFunc(render_scene)
        elif cb == "idle":
            glutIdleFunc(update_animation)
        elif cb == "keyboard":
            glutKeyboardFunc(handle_keyboard)
        elif cb == "special":
            glutSpecialFunc(handle_special_keys)

    print("=" * 50)
    print("RAINFALL HOUSE ANIMATION - CONTROLS")
    print("=" * 50)
    print("LEFT ARROW   : Bend rain to the left")
    print("RIGHT ARROW  : Bend rain to the right")
    print("D key        : Transition to Day (lighter)")
    print("N key        : Transition to Night (darker)")
    print("=" * 50)

    glutMainLoop()

#run only when script is executed directly
if __name__ == "__main__":
    main()
