# Week-12-Draft it is in makeacode 

#Code for my project 

# ===== SETUP =====
# Import the random integer function for potential future use
from random import randint

# Define custom sprite kinds to categorize different game objects
@namespace
class SpriteKind:
    coin = SpriteKind.create()  # Custom kind for coins (not currently used)

@namespace
class SpriteKind:
    TextBubble = SpriteKind.create()  # Custom kind for text bubble sprites

# ===== COLOUR VALUES =====
# Define colour constants using MakeCode Arcade's colour palette indices
BLACK = 15  # Colour index for black
WHITE = 1   # Colour index for white
GREEN = 7   # Colour index for green

# ===== TIMING CONSTANTS =====
TEXT_BUBBLE_DURATION = 2000  # Duration text bubbles display (2000 milliseconds = 2 seconds)

# ===== HELPER FUNCTIONS =====
def text_width(text: string):
    """
    Calculates the pixel width needed for a text string.
    Each character is approximately 6 pixels wide, plus 10 pixels for padding.
    
    Args:
        text: The string to measure
    Returns:
        Integer representing pixel width needed
    """
    return len(text) * 6 + 10

# ===== INITIALIZE LEVEL =====
# Create a custom background image (160x120 pixels - the screen size)
bg = image.create(160, 120)
bg.fill(GREEN)  # Fill the entire background with green colour
scene.set_background_image(bg)  # Set this as the active background

# Load and set the tilemap (defines the road/path layout)
tiles.set_current_tilemap(tilemap("BACKGROUND_ROAD"))

# ===== PLAYER CHARACTER SETUP =====
# Create the player character sprite
character = sprites.create(assets.image("PLAYER"), SpriteKind.player)
character.set_position(16, 60)  # Start position (x=16, y=60)
controller.move_sprite(character)  # Enable D-pad control of the character
scene.camera_follow_sprite(character)  # Camera follows player as they move
info.set_score(0)  # Initialize score at 0
character.z = 100  # Set z-index to 100 (renders above most other sprites)

# ===== ACCESSIBILITY TOGGLES =====
# Global state variables to track accessibility settings
dark_mode = False  # Tracks if dark mode is enabled (starts disabled)
large_text = False  # Tracks if large text mode is enabled (starts disabled)
current_bubble: Sprite = None  # Stores reference to the currently displayed text bubble

# ===== DARK MODE TOGGLE FUNCTION =====
def toggle_dark_mode():
    """
    Toggles dark mode on/off by changing the background colour.
    When enabled: background becomes black (high contrast)
    When disabled: background returns to green
    """
    global dark_mode  # Access the global dark_mode variable
    dark_mode = not dark_mode  # Flip the boolean value (True becomes False, False becomes True)
    
    # Change background based on current state
    if dark_mode:
        bg.fill(BLACK)  # Fill background with black
    else:
        bg.fill(GREEN)  # Fill background with green
    
    # Provide user feedback showing current state
    game.show_long_text("Dark mode: " + ("ON" if dark_mode else "OFF"), DialogLayout.BOTTOM)

# ===== LARGE TEXT TOGGLE FUNCTION =====
def toggle_text():
    """
    Toggles large text mode on/off.
    When enabled: text bubbles and headers become larger and reposition
    When disabled: returns to normal size
    """
    global large_text  # Access the global large_text variable
    large_text = not large_text  # Flip the boolean value
    
    # Provide user feedback showing current state
    game.show_long_text("Larger text: " + ("ON" if large_text else "OFF"), DialogLayout.BOTTOM)
    
    # Redraw all category headers with appropriate sizing
    if large_text:
        draw_headers(True)  # Draw headers in large mode
    else:
        draw_headers(False)  # Draw headers in normal mode

# ===== CONTROLLER EVENT BINDINGS =====
# Register button press events to trigger accessibility functions
controller.B.on_event(ControllerButtonEvent.PRESSED, toggle_dark_mode)  # B button = dark mode
controller.A.on_event(ControllerButtonEvent.PRESSED, toggle_text)  # A button = large text

# ===== CUSTOM LARGE TEXT BUBBLE SYSTEM =====
def big_say(target: Sprite, text: str):
    """
    Creates a custom, scalable text bubble that follows a sprite.
    This replaces the built-in say_text() function to provide larger, more accessible text.
    
    Args:
        target: The sprite the bubble should follow (usually the character)
        text: The text to display in the bubble
    """
    global current_bubble  # Access global variable to track active bubble
    
    # STEP 1: Clean up any existing bubble
    if current_bubble:
        current_bubble.destroy()  # Remove the old bubble from the game
        current_bubble = None  # Clear the reference
    
    # STEP 2: Build the bubble image dynamically
    bubble_img = image.create(text_width(text), 16)  # Create image with calculated width, 16px height
    bubble_img.fill(WHITE)  # Fill with white background
    bubble_img.draw_rect(0, 0, text_width(text), 16, BLACK)  # Draw black border around edges
    bubble_img.print(text, 4, 4, BLACK)  # Print text at position (4,4) in black colour
    
    # STEP 3: Convert image to sprite
    bubble = sprites.create(bubble_img, SpriteKind.TextBubble)  # Create sprite from image
    bubble.set_scale(1.5)  # Scale to 150% size (makes it 50% larger)
    bubble.z = 100  # Set z-index to 100 (renders above most elements)
    
    # STEP 4: Store reference and capture start time
    current_bubble = bubble  # Save to global variable for tracking
    start_time = game.runtime()  # Record current time in milliseconds
    
    # STEP 5: Define update function that runs every frame
    def updater():
        """
        Update function called every frame (~60 times per second).
        Handles bubble positioning and automatic destruction after timeout.
        """
        global current_bubble  # Access global variable
        
        # Safety check: ensure bubble and target still exist
        if not bubble or not target:
            return
        
        # Position bubble above the target sprite
        bubble.set_position(
            target.x,  # Same x position as target
            target.y - target.height // 2 - (16 * 1.5) // 2 - 2  # Above target's head
            # Calculation breakdown:
            # - target.y: target's center y position
            # - target.height // 2: half of target's height (gets to top edge)
            # - (16 * 1.5) // 2: half of scaled bubble height
            # - 2: additional 2-pixel gap
        )
        
        # Check if the bubble has exceeded its display duration
        if game.runtime() - start_time > TEXT_BUBBLE_DURATION:
            bubble.destroy()  # Remove bubble from game
            
            # Only clear global reference if this is still the current bubble
            if current_bubble == bubble:
                current_bubble = None
            
            # Stop this update function by replacing it with an empty one
            game.on_update(empty_updater)
    
    # Empty function to replace updater after bubble timeout
    def empty_updater():
        pass
    
    # Register the updater function to run every frame
    game.on_update(updater)

# ===== CATEGORY HEADER RENDERING =====
def header_at(x, y, text, big: bool = False):
    """
    Creates a category header banner at specified coordinates.
    Headers are styled boxes with text that label each communication category.
    
    Args:
        x: X position for the header
        y: Y position for the header
        text: Text to display in the header
        big: Whether to scale the header (checks global large_text variable)
    """
    global large_text  # Access global variable to check text size setting
    
    # Create image for the header banner
    b = image.create(len(text) * 6 + 8, 13)  # Width based on text length, 13px height
    b.fill(14)  # Fill with colour index 14 (light grey/brown)
    b.draw_rect(0, 0, b.width - 1, b.height - 1, 1)  # Draw white border
    b.print(text, 3, 2, 1)  # Print text at (3,2) in white
    
    # Convert to sprite
    banner = sprites.create(b, SpriteKind.enemy)  # Using 'enemy' kind to identify headers
    banner.set_position(x, y)  # Position the banner
    
    # Scale up if large text mode is enabled
    if large_text:
        banner.set_scale(2)  # Double the size

# ===== HELPER FUNCTION TO DRAW ALL HEADERS =====
def draw_headers(big: bool = False):
    """
    Draws all category headers on the screen.
    Called when large text mode is toggled to reposition headers appropriately.
    
    Args:
        big: Whether headers should be drawn in large mode
    """
    global large_text  # Access global variable
    
    # STEP 1: Clear all existing headers
    for headers in sprites.all_of_kind(SpriteKind.enemy):
        headers.destroy()  # Remove from game
    
    # STEP 2: Draw new headers with appropriate positioning
    if large_text:
        # Positions for large text mode (headers need more space)
        header_at(140, 95, "EMOTIONS")
        header_at(171, 193, "COMMUNICATION")
        header_at(177, 291, "COMMON PHRASES")
        header_at(177, 390, "COMMON PHRASES")
    else:
        # Positions for normal text mode
        header_at(125, 96, "EMOTIONS")
        header_at(140, 195, "COMMUNICATION")
        header_at(142, 293, "COMMON PHRASES")
        header_at(142, 392, "COMMON PHRASES")

# Draw initial headers when game starts
draw_headers(False)

# ===== COMMUNICATION BOX LAYOUT SYSTEM =====

# Helper function to create a single communication box
def make_box(x: int, y: int, label: str, icon_img):
    """
    Creates a communication box with an icon and stores its label.
    Each box consists of two sprites: a frame and an icon inside it.
    
    Args:
        x: X position for the box
        y: Y position for the box
        label: Text label (stored as metadata, displayed when touched)
        icon_img: Image to display as the icon
    Returns:
        The box sprite
    """
    # Create the box frame sprite using predefined image
    box = sprites.create(img("""
        f f f f f f f f f f f f f f f f f f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f . . . . . . . . . . . . . . . . f
        f f f f f f f f f f f f f f f f f f
    """), SpriteKind.food)  # Using 'food' kind for communication boxes
    
    box.set_position(x, y)  # Position the box
    box.image.replace(15, 15)  # Replace colour 15 with 15 (draws the border properly)
    
    # Create the icon sprite (displays inside the box)
    icon = sprites.create(icon_img, SpriteKind.projectile)  # Using 'projectile' kind for icons
    icon.set_position(x, y)  # Position at same location as box (centered)
    icon.z = box.z + 1  # Place icon one layer above the box (so it's visible on top)
    
    # Store the label as metadata on the box sprite
    sprites.set_data_string(box, "label", label)  # Key: "label", Value: the text string
    
    return box  # Return the box sprite

# Helper function to create an entire row of boxes
def make_row(start_x: int, y: int, x_offset: int, items: list[dict]):
    """
    Generates multiple boxes in a horizontal row using procedural generation.
    
    Args:
        start_x: Starting x position for the first box
        y: Y position for all boxes in this row
        x_offset: Horizontal spacing between boxes (in pixels)
        items: List of dictionaries, each containing 'label' and 'icon' keys
    """
    i = 0  # Counter for current box position
    for it in items:  # Iterate through each item in the list
        # Calculate x position: start_x + (box_number * spacing)
        # Example: 105 + (0*40)=105, 105 + (1*40)=145, 105 + (2*40)=185, etc.
        make_box(start_x + i * x_offset, y, it["label"], it["icon"])
        i += 1  # Increment counter for next box

# ===== COMMUNICATION CONTENT DATA STRUCTURES =====

# Row 1: Emotions
# Data structure: List of dictionaries
# Each dictionary has two keys: 'label' (text) and 'icon' (image)
emotions = [
    {"label": "Happy", "icon": assets.image("ICON_SMILE")},
    {"label": "Sad", "icon": assets.image("ICON_SAD")},
    {"label": "Angry", "icon": assets.image("ICON_ANGRY")},
    {"label": "Excited", "icon": assets.image("ICON_EXCITED")},
    {"label": "Confused", "icon": assets.image("ICON_CONFUSED")},
    {"label": "Not Sure", "icon": assets.image("ICON_NOT_SURE")},
]
# Generate the emotions row: starts at x=105, y=121, boxes spaced 40px apart
make_row(105, 121, 40, emotions)

# Row 2: Basic Communication
communication = [
    {"label": "Yes", "icon": assets.image("ICON_TICK")},
    {"label": "No", "icon": assets.image("ICON_CROSS")},
    {"label": "Thank you", "icon": assets.image("ICON_TY")},
    {"label": "Please", "icon": assets.image("ICON_PLEASE")},
    {"label": "Help", "icon": assets.image("ICON_HELP")},
    {"label": "Stop", "icon": assets.image("ICON_STOP")}
]
# Generate the communication row: starts at x=105, y=220, boxes spaced 40px apart
make_row(105, 220, 40, communication)

# Row 3: Common Phrases Part 1
phrases1 = [
    {"label": "How are you?", "icon": assets.image("ICON_SMILE")},
    {"label": "I'm good, thanks", "icon": assets.image("ICON_SMILE")},
    {"label": "Bathroom", "icon": assets.image("ICON_TOILET")},
    {"label": "Im uncomfortable", "icon": assets.image("ICON_ANGRY")},
    {"label": "I need a break", "icon": assets.image("ICON_ANGRY")},
    {"label": "I'm hurt", "icon": assets.image("ICON_SMILE")},
]
# Generate phrases row 1: starts at x=105, y=320, boxes spaced 40px apart
make_row(105, 320, 40, phrases1)

# Row 4: Common Phrases Part 2
phrases2 = [
    {"label": "I'm hungry", "icon": assets.image("ICON_FOOD")},
    {"label": "I'm tired", "icon": assets.image("ICON_TIRED")},
    {"label": "I feel sick", "icon": assets.image("ICON_SICK")},
    {"label": "My tummy hurts", "icon": assets.image("ICON_HUNGRY")},
    {"label": "I'm thirsty", "icon": assets.image("ICON_WATER")},
    {"label": "What's the time?", "icon": assets.image("ICON_CLOCK")},
]
# Generate phrases row 2: starts at x=105, y=420, boxes spaced 40px apart
make_row(105, 420, 40, phrases2)

# ===== COLLISION DETECTION SYSTEM =====

# Constant defining cooldown period
COOLDOWN_MS = 600  # 600 milliseconds (0.6 seconds) between triggers

# Function to check if enough time has passed since last trigger
def can_trigger(box):
    """
    Checks if a box is ready to be triggered again.
    Prevents spam/accidental repeated activations.
    
    Args:
        box: The sprite to check
    Returns:
        Boolean: True if enough time has passed, False otherwise
    """
    # Retrieve the last trigger time from sprite data (defaults to 0 if never triggered)
    last = sprites.read_data_number(box, "lastHit") if sprites.read_data_number(box, "lastHit") else 0
    now = game.runtime()  # Get current time in milliseconds
    
    # Return True if current time minus last trigger time exceeds cooldown period
    return (now - last) > COOLDOWN_MS

# Function to mark a box as just triggered
def mark_triggered(box):
    """
    Records the current time as the last trigger time for this box.
    
    Args:
        box: The sprite to mark
    """
    sprites.set_data_number(box, "lastHit", game.runtime())  # Store current timestamp

# Main collision handler function
def on_overlap(sprite, other):
    """
    Called automatically when the player sprite overlaps with a communication box.
    Handles all interaction logic: cooldown check, displaying text, audio feedback,
    visual effects, score updates, and rewards.
    
    Args:
        sprite: The player character (automatically provided by event system)
        other: The box that was touched (automatically provided by event system)
    """
    # STEP 1: Check cooldown - ignore if box was recently triggered
    if not can_trigger(other):
        return  # Exit function early (early return pattern)
    
    # STEP 2: Mark this box as triggered (start its cooldown)
    mark_triggered(other)
    
    # STEP 3: Increase score by 1
    info.change_score_by(1)
    
    # STEP 4: Retrieve the label text stored in this box
    label = sprites.read_data_string(other, "label")
    
    # STEP 5: Display text bubble (method depends on accessibility setting)
    global large_text  # Access global variable
    if large_text:
        big_say(character, label)  # Use custom large text system
    else:
        character.say_text(label, TEXT_BUBBLE_DURATION)  # Use built-in system
    
    # STEP 6: Play audio feedback
    music.ba_ding.play()  # Short "ding" sound
    
    # STEP 7: Show visual effect on the box
    other.start_effect(effects.hearts, 200)  # Heart particles for 200ms
    
    # STEP 8: Nudge character away from box to prevent immediate re-trigger
    # This is a bug prevention technique - moving the character slightly
    # ensures they don't stay overlapping with the box
    if character.x < other.x:
        character.x -= 10  # Box is to the right, push character left
    else:
        character.x += 10  # Box is to the left, push character right
    
    if character.y < other.y:
        character.y -= 10  # Box is below, push character up
    else:
        character.y += 10  # Box is above, push character down
    
    # STEP 9: Check for rewards at score milestones
    s = info.score()  # Get current score
    if s == 15:
        game.show_long_text("Reward: you get a chocolate!", DialogLayout.BOTTOM)
    elif s == 25:
        game.show_long_text("Reward: you get a 10 minute break!", DialogLayout.BOTTOM)
    elif s == 35:
        game.show_long_text("Reward: you get a 15 minute break!", DialogLayout.BOTTOM)

# Register the overlap event handler
# When a sprite of kind 'player' overlaps with a sprite of kind 'food', call on_overlap
sprites.on_overlap(SpriteKind.player, SpriteKind.food, on_overlap)

# ===== INITIAL GAME PROMPT =====
# Display instructions when the game starts
game.show_long_text("Move along the path with the D-pad.\nTouch a box to say it!",
    DialogLayout.BOTTOM)
