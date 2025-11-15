from OpenGL.GL import *
from OpenGL.GLUT import *
from OpenGL.GLU import *
import random

# Global variables
WINDOW_WIDTH = 800
WINDOW_HEIGHT = 600

# Point class (properties of point: position, direction, color)
class Point:
    def __init__(self, x, y, dx, dy, r, g, b):
        self.x = x
        self.y = y
        self.dx = dx
        self.dy = dy
        self.r = r
        self.g = g
        self.b = b

    def draw(self):
        glPointSize(10.0)
        glBegin(GL_POINTS)
        
        # Blinking logic
        if is_blinking:
            if blink_timer < 500:
                glColor3f(0.0, 0.0, 0.0)  # if less than 500, blink black
            else:
                glColor3f(self.r, self.g, self.b)
        else:
            glColor3f(self.r, self.g, self.b)
        
        glVertex2f(self.x, self.y)  # Draw point at x, y coord
        glEnd()

points = []

# Global settings
speed = 0.01  # speed of the points
is_frozen = False
is_blinking = False
blink_timer = 0

def convert_coordinate(x, y):  # change coord for generating random points
    # Convert screen coord ---> OpenGL coord
    a = (x / WINDOW_WIDTH) * 500 - 250  # horizontal pos where 0 is the center of window
    b = 250 - (y / WINDOW_HEIGHT) * 500  # vertical pos. top bottom edges are 250, -250
    return a, b

def init():
    glClearColor(0.0, 0.0, 0.0, 1.0)
    glMatrixMode(GL_PROJECTION)
    glLoadIdentity()
    glOrtho(-250, 250, -250, 250, 0, 1)
    glMatrixMode(GL_MODELVIEW)

def draw_point(point_data):
    point_data.draw()

def display():
    """Display callback function."""
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)  # clears the screen before drawing anything new
    glLoadIdentity()  # resets transformation matrix

    # Goes through each point of the points list and calls the function
    [draw_point(point) for point in points]

    # After drawing all in back buffer, it swaps to front so that it is visible
    glutSwapBuffers()

def animate():  # move, bounce, stay within range
    """Update callback for animation - called continuously."""
    global is_blinking, blink_timer
    
    # Update points only if not frozen
    if not is_frozen:
        # Loop through list and in each iteration get index and value
        for i, point in enumerate(points):
            # Update position
            point.x += point.dx * speed
            point.y += point.dy * speed
            
            # Handles bounce
            point.dx = -point.dx if point.x < -250 or point.x > 250 else point.dx
            point.dy = -point.dy if point.y < -250 or point.y > 250 else point.dy
            
            # Stays within the window range
            point.x = max(min(point.x, 250), -250)
            point.y = max(min(point.y, 250), -250)
    
    blink_timer = (blink_timer + 1) if is_blinking else blink_timer
    blink_timer = blink_timer % 1000 if is_blinking else blink_timer  # The timer loops every 1000 calls
    
    glutPostRedisplay()  # Re-render screen

def keyboard_listener(key, x, y):
    """Keyboard callback function."""
    global is_frozen
    
    if key == b' ':  # Freeze using space bar
        # true-->false; unfreeze, false-->true; freeze
        is_frozen = not is_frozen if is_frozen else not is_frozen
        print("Frozen: " + str(is_frozen))

def special_key_listener(key, x, y):  # up down key for speed
    """Special keys callback function."""
    global speed
    
    speed = speed * 2 if key == GLUT_KEY_UP else speed / 2 if key == GLUT_KEY_DOWN else speed
    
    if key == GLUT_KEY_UP:
        print("Speed increased: " + str(speed))
    elif key == GLUT_KEY_DOWN:
        print("Speed decreased: " + str(speed))

def mouse_listener(button, state, x, y):  # right click to generate random points
    """Mouse callback function."""
    global is_blinking, blink_timer
    
    if button == GLUT_RIGHT_BUTTON and state == GLUT_DOWN:
        gl_x, gl_y = convert_coordinate(x, y)
        
        # Random direction
        dx = -1.0 if random.random() < 0.5 else 1.0
        dy = -1.0 if random.random() < 0.5 else 1.0
        
        # Random color
        r = random.random()
        g = random.random()
        b = random.random()
        
        # Create new point as a Point object
        point = Point(gl_x, gl_y, dx, dy, r, g, b)
        points.append(point)
        
        print("Point created at OpenGL coordinates: " + str(gl_x) + ", " + str(gl_y))
    
    elif button == GLUT_LEFT_BUTTON and state == GLUT_DOWN:  # left button for blink
        if is_blinking:
            is_blinking = False
            blink_timer = 0
        else:
            is_blinking = True
        
        print("Blinking: " + str(is_blinking))

def main():
    """Main function."""
    glutInit()
    glutInitDisplayMode(GLUT_RGBA)
    glutInitWindowSize(WINDOW_WIDTH, WINDOW_HEIGHT)
    glutInitWindowPosition(100, 100)
    glutCreateWindow(b"Amazing Box")
    
    init()
    
    glutDisplayFunc(display)
    glutIdleFunc(animate)
    glutKeyboardFunc(keyboard_listener)
    glutSpecialFunc(special_key_listener)
    glutMouseFunc(mouse_listener)
    
    glutMainLoop()

if __name__ == "__main__":
    main()
