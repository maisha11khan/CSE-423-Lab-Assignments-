from OpenGL.GL import *
from OpenGL.GLUT import *
from OpenGL.GLU import *
import random
import time
window_config = {
    'width': 500,
    'height': 500
}
WINDOW_WIDTH = window_config['width']
WINDOW_HEIGHT = window_config['height']

# Game object dimensions 
object_dimensions = (
    ('catcher', 80, 20),
    ('diamond', 20, 20)
)
CATCHER_WIDTH = object_dimensions[0][1]
CATCHER_HEIGHT = object_dimensions[0][2]
DIAMOND_SIZE = object_dimensions[1][1]

# Catcher state variables 

catcher_position = [0, -200]
catcher_x = catcher_position[0]
catcher_y = catcher_position[1]
catcher_speed = 200

# ===== Diamond State Variables =====

diamond_initial_state = {
    'x': 0,
    'y': 200,
    'speed': 50,
    'acceleration': 10
}
diamond_x = diamond_initial_state['x']
diamond_y = diamond_initial_state['y']
diamond_speed = diamond_initial_state['speed']
diamond_acceleration = diamond_initial_state['acceleration']

# Diamond color 
diamond_color = [1.0, 1.0, 0.0]
# Boolean flags stored 
game_flags = (0, False, False, False)
score = game_flags[0]
game_over = game_flags[1]
paused = game_flags[2]
cheat_mode = game_flags[3]

#reference for calculatong delta time
#stores current time in last_time
last_time = time.time()

# Buttons config
button_configs = [
    ('restart', -220, 220, 20),
    ('pause', 0, 220, 20),
    ('exit', 220, 220, 20)
]
restart_button = (button_configs[0][1], button_configs[0][2], button_configs[0][3])
pause_button = (button_configs[1][1], button_configs[1][2], button_configs[1][3])
exit_button = (button_configs[2][1], button_configs[2][2], button_configs[2][3])

#  Absolute Value Calculator
def calculate_absolute(value):
    return value if value >= 0 else -value

#  Zone classification (if line is steep or gentle based on slope)
#steep--->handles vertical movement
#gentle--->handles horizontal movement (algo will differ depending on slope)
def classify_by_slope(abs_dx, abs_dy):
    return abs_dy > abs_dx

# Quadrant Checker  
#classifies the direction of line seeing x,y
def check_quadrant(dx_positive, dy_positive):
    return (dx_positive, dy_positive)

#  Zone Finding System
#implements: 8 way symmetry
def find_zone(dx, dy):
    abs_dx = calculate_absolute(dx)
    abs_dy = calculate_absolute(dy)
    
    is_steep = classify_by_slope(abs_dx, abs_dy)
    
    quadrant_info = check_quadrant(dx > 0, dy > 0)
    dx_pos = quadrant_info[0]
    dy_pos = quadrant_info[1]
    
    zone = 0
    if not is_steep:
        if dx_pos and dy_pos:
            zone = 0
        elif not dx_pos and dy_pos:
            zone = 3
        elif not dx_pos and not dy_pos:
            zone = 4
        else:
            zone = 7
    else:
        if dx_pos and dy_pos:
            zone = 1
        elif not dx_pos and dy_pos:
            zone = 2
        elif not dx_pos and not dy_pos:
            zone = 5
        else:
            zone = 6
    
    return zone


#Zone-to-Zone0 conversion
#lambda-->creates small funcs that apply coordinate transformations based on the line zone
def get_zone_transform(zone):
    
    zone_transforms = {
        0: lambda px, py: (px, py),
        1: lambda px, py: (py, px),
        2: lambda px, py: (py, -px),
        3: lambda px, py: (-px, py),
        4: lambda px, py: (-px, -py),
        5: lambda px, py: (-py, -px),
        6: lambda px, py: (-py, px),
        7: lambda px, py: (px, -py)
    }
    return zone_transforms.get(zone)


#Zone Conversion 
def convert_to_zone0(x, y, zone):
    transform_func = get_zone_transform(zone)
    result = transform_func(x, y)
    return result

#conversion from Zone 0 back to original zone
def convert_from_zone0(x, y, zone):
    if zone <= 3:
        if zone == 0:
            return (x, y)
        elif zone <= 2:
            if zone == 1:
                return (y, x)
            else:
                return (-y, x)
        else:
            return (-x, y)
    else:
        if zone == 4:
            return (-x, -y)
        elif zone <= 6:
            if zone == 5:
                return (-y, -x)
            else:
                return (y, -x)
        else:
            return (x, -y)


# Calculates initial decision parameter for MPL
def calculate_initial_decision(dy, dx):
   return 2 * dy - dx


#  Calculate E and NE increment values
def calculate_increments(dy, dx):
 
    incE = 2 * dy
    incNE = 2 * (dy - dx)
    return (incE, incNE)


# Midpoint algo for zone 0
def midpoint_line_zone0(x1, y1, x2, y2):
  
    dx = x2 - x1
    dy = y2 - y1
    
    d = calculate_initial_decision(dy, dx)
    
    increments = calculate_increments(dy, dx)
    incE = increments[0]
    incNE = increments[1]
    
    y = y1 #line starts on the vertical axis
    points = []
    
    
    x_values = []
    temp_x = x1 #starting x coord of the line
    while temp_x <= x2:
        x_values.append(temp_x)
        temp_x = temp_x + 1
    
  
    idx = 0
    current_y = y1
    current_d = d
    while idx < len(x_values):
        points.append((x_values[idx], current_y))
        
        if current_d > 0:
            current_d = current_d + incNE
            current_y = current_y + 1
        else:
            current_d = current_d + incE
        
        idx = idx + 1
    
    return points


# Rounds floating point coordinates to integers
def round_coordinates(x1, y1, x2, y2):
   
    coords = (x1, y1, x2, y2)
    #takes the coordinate, rounds them to the nearest whole number, stores as a new tuple.
    rounded = tuple(int(c + 0.5) for c in coords)
    return rounded


# Main mid point line algo 
#Complete line drawing with zone conversion
#only gl_point 
def draw_line_midpoint(x1, y1, x2, y2):
   
    rounded = round_coordinates(x1, y1, x2, y2)
    x1, y1, x2, y2 = rounded
    #helps to determine slope of line
    dx = x2 - x1
    dy = y2 - y1
    
    zone = find_zone(dx, dy)
    
    start_converted = convert_to_zone0(x1, y1, zone)
    end_converted = convert_to_zone0(x2, y2, zone)
    
    points_zone0 = midpoint_line_zone0(start_converted[0], start_converted[1], 
                                        end_converted[0], end_converted[1])
    
    glBegin(GL_POINTS)
    point_index = 0
    total_points = len(points_zone0)
    while point_index < total_points:
        current_point = points_zone0[point_index]
        #converts the point back from Zone 0 to its original zone
        original = convert_from_zone0(current_point[0], current_point[1], zone)
        glVertex2f(original[0], original[1])
        point_index = point_index + 1
    glEnd()


#catcher color
def get_catcher_color(is_game_over):
    
    if is_game_over:
        return (1.0, 0.0, 0.0)
    else:
        return (1.0, 1.0, 1.0)


#catcher with 4 mpl
def draw_catcher():
 
    color = get_catcher_color(game_over)
    glColor3f(color[0], color[1], color[2])
    #catcher is at center horizontally based on its width
    half_width = CATCHER_WIDTH // 2
    
    vertices = []
    #horizontal line at top 
    vertices.append((catcher_x - half_width, catcher_y, 
                    catcher_x + half_width, catcher_y))
    #diagonal line from the top left to the bottom left
    vertices.append((catcher_x - half_width, catcher_y, 
                    catcher_x - half_width + 10, catcher_y + CATCHER_HEIGHT))
    #diagonal line from the top right to the bottom right
    vertices.append((catcher_x + half_width, catcher_y, 
                    catcher_x + half_width - 10, catcher_y + CATCHER_HEIGHT))
    #bottom horizontal edge
    vertices.append((catcher_x - half_width + 10, catcher_y + CATCHER_HEIGHT, 
                    catcher_x + half_width - 10, catcher_y + CATCHER_HEIGHT))
    #draws line by mpl
    line_num = 0
    while line_num < len(vertices):
        v = vertices[line_num]
        draw_line_midpoint(v[0], v[1], v[2], v[3])
        line_num = line_num + 1

#diamond with 4 mpl
def draw_diamond():
   
    glColor3f(diamond_color[0], diamond_color[1], diamond_color[2])
    #distance from the center to edge
    half_size = DIAMOND_SIZE // 2
    #top,right,bottom,left
    offsets = [(0, half_size), (half_size, 0), (0, -half_size), (-half_size, 0)]
    vertices = []
    #calculates abs pos adding offest(shows distance from center to each 4 point)
    for offset in offsets:
        vertices.append((diamond_x + offset[0], diamond_y + offset[1]))
    
    vertex_idx = 0
    while vertex_idx < len(vertices):
        next_idx = (vertex_idx + 1) % len(vertices)
        draw_line_midpoint(vertices[vertex_idx][0], vertices[vertex_idx][1],
                          vertices[next_idx][0], vertices[next_idx][1])
        vertex_idx = vertex_idx + 1


def draw_restart_button():
  
    glColor3f(0.0, 0.8, 0.8)
    x, y, s = restart_button
    
    half_s = s // 2
    #forms upper part,lower part,right side
    segments = [
        ((x + s, y + half_s), (x, y)),
        ((x, y), (x + s, y - half_s)),
        ((x + s, y + half_s), (x + s, y - half_s))
    ]
    
    seg_idx = 0
    while seg_idx < len(segments):
        start, end = segments[seg_idx]
        draw_line_midpoint(start[0], start[1], end[0], end[1]) #using mpl
        seg_idx = seg_idx + 1


def draw_pause_button():
    glColor3f(1.0, 0.75, 0.0)
    x, y, s = pause_button
    half_s = s // 2 #position the triangle 
    third_s = s // 3#position the bars (pause)
    button_state = 'play' if paused else 'pause'
    
    if button_state == 'play':
        triangle_points = [
            (x - half_s, y + half_s), #left vertex
            (x + half_s, y),#right 
            (x - half_s, y - half_s) #bottom
        ]
        #draws each side of the triangle by connecting each point to the next looping back to the start
        point_count = len(triangle_points)
        i = 0
        while i < point_count:
            next_i = (i + 1) % point_count
            p1 = triangle_points[i]
            p2 = triangle_points[next_i]
            draw_line_midpoint(p1[0], p1[1], p2[0], p2[1]) #using mpl
            i = i + 1
    else:
        bars = [
            (x - third_s, y + half_s, x - third_s, y - half_s), #left bar
            (x + third_s, y + half_s, x + third_s, y - half_s) #right bar
        ]
        
        for bar in bars:
            draw_line_midpoint(bar[0], bar[1], bar[2], bar[3])


def draw_exit_button():
  
    glColor3f(1.0, 0.0, 0.0)
    x, y, s = exit_button
    
    half_s = s // 2
    
    diagonals = {
        'main': (x - half_s, y + half_s, x + half_s, y - half_s),
        'anti': (x - half_s, y - half_s, x + half_s, y + half_s)
    }
    
    for key in diagonals:
        line_data = diagonals[key]
        draw_line_midpoint(line_data[0], line_data[1], line_data[2], line_data[3]) #using mpl


# checks 2 range overlap
#implements helper for AABB collision detection
def check_range_overlap(min1, max1, min2, max2):
    return not (max1 < min2 or min1 > max2)


# Collision Detection (Axis-Aligned Bounding Box collision)
def check_collision():
   
    diamond_bounds = (
        diamond_x - DIAMOND_SIZE // 2, #left
        diamond_x + DIAMOND_SIZE // 2, #right
        diamond_y - DIAMOND_SIZE // 2, #bottom
        diamond_y + DIAMOND_SIZE // 2 #top
    )
    
    catcher_bounds = (
        catcher_x - CATCHER_WIDTH // 2, #left
        catcher_x + CATCHER_WIDTH // 2,#right
        catcher_y,#bottom
        catcher_y + CATCHER_HEIGHT #top
    )
    
    d_left, d_right, d_bottom, d_top = diamond_bounds
    c_left, c_right, c_bottom, c_top = catcher_bounds
    #Checks if x axis of the diamond and catcher overlap
    x_overlap = check_range_overlap(d_left, d_right, c_left, c_right)
    ##Checks if y axis of the diamond and catcher overlap
    y_overlap = check_range_overlap(d_bottom, d_top, c_bottom, c_top)
    
    return x_overlap and y_overlap


# color generator
def generate_random_color():
  
    color = []
    channel = 0
    while channel < 3:
        color.append(random.uniform(0.5, 1.0))#generates random flt num between this range
        channel = channel + 1
    return color


# Game logic 
#generates diamond at random position,color
def spawn_diamond():
   
    global diamond_x, diamond_y, diamond_color
    #generates random value each time
    random.seed()
    diamond_x = random.randint(-200, 200)
    diamond_y = 250
    
    diamond_color = generate_random_color()


def reset_game():
  
    global score, game_over, diamond_speed, catcher_x, paused, cheat_mode, last_time
    
    state_values = (0, False, False, False, 50, 0)
    score, game_over, paused, cheat_mode, diamond_speed, catcher_x = state_values
    #tracks elapsed time in game and ensures that time starts fresh when the game is reset
    last_time = time.time()
    
    spawn_diamond()
    print("Starting Over")

#inc score and increse speed
def handle_catch():
   
    global score, diamond_speed
    
    score = score + 1
    diamond_speed = diamond_speed + diamond_acceleration
    print(f"Score: {score}")
    spawn_diamond()

#game over state(miss diamond)
def handle_miss():
  
    global game_over
    
    game_over = True
    print(f"Game Over! Final Score: {score}")


# Converts screen coordinates to opengl coordinates
def convert_coordinate(x, y):
   
    offset_x = WINDOW_WIDTH / 2
    offset_y = WINDOW_HEIGHT / 2
    
    gl_coords = (x - offset_x, offset_y - y)
    return gl_coords

#handles button click(start, oause, cross)
def mouse_listener(button, state, x, y):
    global paused, last_time
    if not (button == GLUT_LEFT_BUTTON and state == GLUT_DOWN):
        return
    #converts the mouse screen coordinates to opengl coordinates
    mx, my = convert_coordinate(x, y)
    #has x,y coord and clickable area(margin)
    button_regions = {
        'restart': {'x': restart_button[0], 'y': restart_button[1], 'margin': 20},
        'pause': {'x': pause_button[0], 'y': pause_button[1], 'margin': 20},
        'exit': {'x': exit_button[0], 'y': exit_button[1], 'margin': 20}
    }
    button_names = ['restart', 'pause', 'exit']
    btn_idx = 0
    clicked_button = None
    
    while btn_idx < len(button_names) and clicked_button is None:
        btn_name = button_names[btn_idx]
        region = button_regions[btn_name]
        #checks the mouse coordinates are within the clickable area of button
        x_in_range = region['x'] - region['margin'] < mx < region['x'] + region['margin']
        y_in_range = region['y'] - region['margin'] < my < region['y'] + region['margin']
        
        if x_in_range and y_in_range:
            clicked_button = btn_name
        
        btn_idx = btn_idx + 1
    
    if clicked_button == 'restart':
        reset_game()
    elif clicked_button == 'pause':
        paused = not paused
        
        last_time = time.time()
    elif clicked_button == 'exit':
        print(f"Goodbye! Final Score: {score}")
        glutLeaveMainLoop()

#implements cheat mode
def keyboard_listener(key, x, y):
  
    global cheat_mode
    
    cheat_key = b'c' #b=bytes
    
    if key == cheat_key:
        cheat_mode = not cheat_mode
        status = "activated" if cheat_mode else "deactivated"
        print(f"Cheat mode {status}")


def special_key_listener(key, x, y):  #Catcher Movement and Control using arrow
   
    global catcher_x
    #Check if the catcher can move
    can_move = not game_over and not paused and not cheat_mode
    if not can_move:
        return
    
    move_amount = 10
    bounds = {    #bounds so that catcher doesnt go out of screen
        'left': -250 + CATCHER_WIDTH // 2,
        'right': 250 - CATCHER_WIDTH // 2
    }
    #left right arrow key
    direction = None
    if key == GLUT_KEY_LEFT:
        direction = 'left'
    elif key == GLUT_KEY_RIGHT:
        direction = 'right'
    #Move the Catcher and Ensure so stays within Bound
    if direction == 'left':
        catcher_x = catcher_x - move_amount
        catcher_x = max(bounds['left'], catcher_x)
    elif direction == 'right':
        catcher_x = catcher_x + move_amount
        catcher_x = min(bounds['right'], catcher_x)


# updated game state
#delta time for speed
#c---movement
#diamond fall and collision
def animate():
    global diamond_y, catcher_x, last_time
    
    #checks the game is either over or paused
    should_pause = game_over or paused
    if should_pause:
        last_time = time.time() #updates last_time to the current
        glutPostRedisplay()
        return
    
    current_time = time.time() #current time
    dt = current_time - last_time #frame-rate independent movement
    last_time = current_time
    
    if cheat_mode: #cheat c=mode implemented
        target = diamond_x
        current = catcher_x
        move_speed = catcher_speed * dt #smooth and consistent movement in different frame rates
        
        diff = target - current
        abs_diff = diff if diff >= 0 else -diff
        
        if abs_diff > 0.1:
            direction_multiplier = 1 if diff > 0 else -1
            movement = min(abs_diff, move_speed)
            catcher_x = current + (movement * direction_multiplier) #updates the position of the catcher
        else:
            catcher_x = target
        
        left_bound = -250 + CATCHER_WIDTH // 2
        right_bound = 250 - CATCHER_WIDTH // 2
        
        if catcher_x < left_bound:
            catcher_x = left_bound
        if catcher_x > right_bound:
            catcher_x = right_bound
    #diamond falling at consistant rate
    diamond_y = diamond_y - (diamond_speed * dt) 
    #collision occur and game over
    collision_occurred = check_collision()
    diamond_missed = diamond_y < -250 
    
    if collision_occurred:
        handle_catch()
    else:
        if diamond_missed:
            handle_miss()
    
    glutPostRedisplay()


# draws all the game objects (buttons, catcher, diamond)
def display():
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)## clears the screen before drawing anything new
    glLoadIdentity() # # resets transformation matrix
    
    glViewport(0, 0, WINDOW_WIDTH, WINDOW_HEIGHT)
    glMatrixMode(GL_PROJECTION)
    glLoadIdentity()
    glOrtho(-250, 250, -250, 250, 0, 1)
    glMatrixMode(GL_MODELVIEW)
    
    render_queue = [
        draw_restart_button,
        draw_pause_button,
        draw_exit_button,
        draw_catcher
    ]
    
    if not game_over:
        render_queue.append(draw_diamond)
    
    task_idx = 0
    while task_idx < len(render_queue):
        render_func = render_queue[task_idx]
        render_func()
        task_idx = task_idx + 1
    
    glutSwapBuffers()


#OpenGL initialization and main loop
def main():
   
    global last_time
    
    glutInit()
    glutInitDisplayMode(GLUT_RGBA | GLUT_DOUBLE) #for smooth rendering
    glutInitWindowSize(WINDOW_WIDTH, WINDOW_HEIGHT)
    glutInitWindowPosition(100, 100)
    glutCreateWindow(b"Catch the Diamonds!")
    
    glClearColor(0.0, 0.0, 0.0, 1.0)
    
    random.seed()
    
    spawn_diamond()
    last_time = time.time()
    
    callbacks = {
        'display': glutDisplayFunc,
        'idle': glutIdleFunc,
        'keyboard': glutKeyboardFunc,
        'special': glutSpecialFunc,
        'mouse': glutMouseFunc
    }
    
    handlers = {
        'display': display,
        'idle': animate,
        'keyboard': keyboard_listener,
        'special': special_key_listener,
        'mouse': mouse_listener
    }
    
    for key in callbacks:
        register_func = callbacks[key]
        handler_func = handlers[key]
        register_func(handler_func)
    
    glutMainLoop()


if __name__ == "__main__":
    main()
