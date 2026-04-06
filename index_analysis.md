# index.html - Detailed Code Analysis

This document provides a comprehensive line-by-line analysis of the 1010 Block Puzzle game implemented in index.html using the Phaser 3 game engine.

---

## HTML Structure (Lines 1-23)

### Lines 1-2: Document Type and Language
```html
<!DOCTYPE html>
<html lang="en">
```
- Standard HTML5 doctype declaration
- Sets language to English for accessibility and SEO

### Lines 3-7: Head Section - Meta Tags
```html
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>1010 Block Puzzle</title>
```
- `charset="UTF-8"`: Ensures proper character encoding
- Viewport meta tag configures responsive behavior:
  - `width=device-width`: Match device width
  - `maximum-scale=1.0, user-scalable=no`: Disable zoom to prevent accidental pinch-zoom during gameplay
- Title sets the browser tab text

### Line 8: Phaser Library Loading
```html
<script src="https://cdn.jsdelivr.net/npm/phaser@3.60.0/dist/phaser.min.js"></script>
```
- Loads Phaser 3 version 3.60.0 from CDN (jsdelivr)
- This is the core game engine powering the entire game

### Lines 9-22: CSS Styles
```html
<style>
    body {
        margin: 0;
        padding: 0;
        background-color: #1a1a2e;
        overflow: hidden;
        /* Prevent touch actions like scrolling to improve game feel */
        touch-action: none; 
    }
    canvas {
        display: block;
        margin: 0 auto;
    }
</style>
```
- **body**: Removes default margins, sets dark background (#1a1a2e), hides overflow to prevent scrollbars, disables touch actions to improve mobile gameplay
- **canvas**: Centers the game canvas horizontally

---

## JavaScript - Game Constants (Lines 27-61)

### Lines 27-32: Grid Configuration Constants
```javascript
const GRID_SIZE = 10;
const CELL_SIZE = 50;
const GRID_X = 50; // Offset from left
const GRID_Y = 150; // Offset from top
const TRAY_Y = 750; // Y position for spawning shapes
```
- `GRID_SIZE`: 10x10 grid (standard 1010 puzzle size)
- `CELL_SIZE`: Each cell is 50 pixels square
- `GRID_X`: Grid positioned 50px from left edge
- `GRID_Y`: Grid positioned 150px from top (leaving room for score)
- `TRAY_Y`: Shape tray positioned at Y=750 (near bottom)

### Lines 34-38: Color Palette
```javascript
const COLORS = [
    0xe94560, 0x0f3460, 0x533483, 0xf9ed69, 0xf08a5d, 
    0xb83b5e, 0x6a2c70, 0x00adb5, 0x3fc1c9, 0xfce38a
];
```
- Array of 10 distinct colors in hexadecimal format
- Used to randomly color shape blocks
- Colors range from warm (reds, oranges, yellows) to cool (blues, purples, teals)

### Lines 40-61: Shape Definitions
```javascript
const SHAPES = [
    [[1]], // 1x1
    [[1,1]], // 1x2
    [[1],[1]], // 2x1
    [[1,1,1]], // 1x3
    [[1],[1],[1]], // 3x1
    [[1,1,1,1]], // 1x4
    [[1],[1],[1],[1]], // 4x1
    [[1,1,1,1,1]], // 1x5
    [[1],[1],[1],[1],[1]], // 5x1
    [[1,1],[1,1]], // 2x2
    [[1,1,1],[1,1,1],[1,1,1]], // 3x3
    [[1,0],[1,1]], // Small L (BL)
    [[0,1],[1,1]], // Small L (BR)
    [[1,1],[1,0]], // Small L (TL)
    [[1,1],[0,1]], // Small L (TR)
    [[1,0,0],[1,0,0],[1,1,1]], // Big L (BL)
    [[0,0,1],[0,0,1],[1,1,1]], // Big L (BR)
    [[1,1,1],[1,0,0],[1,0,0]], // Big L (TL)
    [[1,1,1],[0,0,1],[0,0,1]]  // Big L (TR)
];
```
- 20 different shapes defined as 2D binary matrices
- 1 = block present, 0 = empty space
- Shape types include:
  - Single blocks (1x1)
  - Lines (1x2, 1x3, 1x4, 1x5 and their vertical variants)
  - Squares (2x2, 3x3)
  - L-shapes (both small 2x2 and big 3x3 variants with 4 rotations each)

---

## GameScene Class (Lines 63-433)

### Lines 63-66: Class Constructor
```javascript
class GameScene extends Phaser.Scene {
    constructor() {
        super('GameScene');
    }
```
- Extends Phaser.Scene to create the main game scene
- Named 'GameScene' for reference

### Lines 68-150: create() Method - Initialization

#### Lines 69-74: Dynamic Texture Generation
```javascript
create() {
    // Generate a rounded block texture dynamically
    const gfx = this.add.graphics();
    gfx.fillStyle(0xffffff, 1);
    gfx.fillRoundedRect(0, 0, CELL_SIZE - 2, CELL_SIZE - 2, 8);
    gfx.generateTexture('block', CELL_SIZE - 2, CELL_SIZE - 2);
    gfx.destroy();
```
- Creates a white rounded rectangle (48x48px with 8px corner radius)
- Generates a reusable texture named 'block'
- Destroys the Graphics object after texture creation (memory management)
- This avoids loading external image assets

#### Lines 76-79: Instance Variables
```javascript
    this.score = 0;
    this.gridData = []; // 2D array of logic
    this.gridBlocks = []; // 2D array of actual sprites on the grid
    this.trayShapes = []; // Shapes currently available at the bottom
```
- `score`: Tracks player score
- `gridData`: Logical representation of the grid (0=empty, 1=filled)
- `gridBlocks`: Visual sprites on the grid for cleanup/animation
- `trayShapes`: Array of currently available shapes in the tray

#### Lines 81-89: Grid Initialization
```javascript
    // Initialize Grid arrays
    for (let r = 0; r < GRID_SIZE; r++) {
        this.gridData[r] = [];
        this.gridBlocks[r] = [];
        for (let c = 0; c < GRID_SIZE; c++) {
            this.gridData[r][c] = 0;
            this.gridBlocks[r][c] = null;
        }
    }
```
- Initializes both grid arrays as 10x10 matrices
- `gridData` filled with 0 (empty)
- `gridBlocks` filled with null (no sprite)

#### Lines 91-101: Background Grid Rendering
```javascript
    // Draw Background Grid
    for (let r = 0; r < GRID_SIZE; r++) {
        for (let c = 0; c < GRID_SIZE; c++) {
            let bgBlock = this.add.sprite(
                GRID_X + c * CELL_SIZE + CELL_SIZE / 2, 
                GRID_Y + r * CELL_SIZE + CELL_SIZE / 2, 
                'block'
            );
            bgBlock.setTint(0x16213e);
        }
    }
```
- Creates 100 background sprite cells
- Positions them in a 10x10 grid pattern
- Applies dark tint (0x16213e) to create the empty grid appearance
- These serve as visual placeholders for empty cells

#### Lines 103-109: Score UI
```javascript
    // Score UI
    this.scoreText = this.add.text(300, 70, 'SCORE: 0', {
        fontSize: '36px',
        fontFamily: 'Arial',
        fontStyle: 'bold',
        color: '#ffffff'
    }).setOrigin(0.5);
```
- Creates score text at position (300, 70) - top center
- 36px bold white Arial text
- `setOrigin(0.5)` centers the text on its position

#### Line 111: Initial Shape Spawn
```javascript
    this.spawnTrayShapes();
```
- Calls method to generate 3 random shapes in the tray

#### Lines 113-149: Drag and Drop Event Handlers

**Drag Start (Lines 114-119):**
```javascript
    this.input.on('dragstart', (pointer, gameObject) => {
        this.children.bringToTop(gameObject);
        // Scale up and move up so the finger doesn't block the view
        gameObject.setScale(1); 
        gameObject.dragOffsetY = -100;
    });
```
- Brings dragged object to front
- Scales to 1x (full size) from 0.6x in tray
- Offsets Y by -100 to lift shape above finger on mobile

**Drag Movement (Lines 121-124):**
```javascript
    this.input.on('drag', (pointer, gameObject, dragX, dragY) => {
        gameObject.x = dragX;
        gameObject.y = dragY + gameObject.dragOffsetY;
    });
```
- Updates position based on pointer with Y offset applied
- Creates the "lifted above finger" effect

**Drag End (Lines 126-149):**
```javascript
    this.input.on('dragend', (pointer, gameObject) => {
        gameObject.setScale(0.6); // Scale back down
        
        // Calculate position relative to grid
        let hitX = gameObject.x - GRID_X;
        let hitY = gameObject.y - GRID_Y;

        // Determine closest grid column and row
        let col = Math.round((hitX - (gameObject.shapeWidth * CELL_SIZE) / 2) / CELL_SIZE);
        let row = Math.round((hitY - (gameObject.shapeHeight * CELL_SIZE) / 2) / CELL_SIZE);

        if (this.canPlace(gameObject.matrix, row, col)) {
            this.placeShape(gameObject, row, col);
        } else {
            // Snap back to tray using the custom start properties
            this.tweens.add({
                targets: gameObject,
                x: gameObject.startX,
                y: gameObject.startY,
                duration: 200,
                ease: 'Back.out'
            });
        }
    });
```
- Scales shape back to 0.6x (tray size)
- Calculates grid position based on drop location
- Adjusts for shape dimensions to find center cell
- Calls `canPlace()` to validate placement
- If valid: places shape; if invalid: animates snap-back to tray

---

### Lines 152-168: spawnTrayShapes() Method
```javascript
spawnTrayShapes() {
    this.trayShapes = [];
    const trayPositions = [100, 300, 500]; // X coordinates for the 3 slots

    for (let i = 0; i < 3; i++) {
        let shape = this.createShape();
        shape.x = trayPositions[i];
        shape.y = TRAY_Y;
        // Use startX and startY instead of originX and originY
        shape.startX = shape.x;
        shape.startY = shape.y;
        shape.setScale(0.6);
        this.trayShapes.push(shape);
    }
    
    this.checkGameOver();
}
```
- Clears existing tray shapes
- Creates 3 shapes at X positions 100, 300, 500
- Stores original position (startX, startY) for snap-back functionality
- Scales to 0.6x for tray display
- Checks for game over after spawning

---

### Lines 170-207: createShape() Method
```javascript
createShape() {
    let matrix = Phaser.Utils.Array.GetRandom(SHAPES);
    let color = Phaser.Utils.Array.GetRandom(COLORS);
    let container = this.add.container(0, 0);
    
    let rows = matrix.length;
    let cols = matrix[0].length;

    container.matrix = matrix;
    container.color = color;
    container.shapeWidth = cols;
    container.shapeHeight = rows;

    // Center the container pivot based on its dimensions
    let offsetX = (cols * CELL_SIZE) / 2;
    let offsetY = (rows * CELL_SIZE) / 2;

    for (let r = 0; r < rows; r++) {
        for (let c = 0; c < cols; c++) {
            if (matrix[r][c] === 1) {
                let block = this.add.sprite(
                    c * CELL_SIZE - offsetX + CELL_SIZE / 2,
                    r * CELL_SIZE - offsetY + CELL_SIZE / 2,
                    'block'
                );
                block.setTint(color);
                container.add(block);
            }
        }
    }

    // Make interactive
    let hitArea = new Phaser.Geom.Rectangle(-offsetX, -offsetY, cols * CELL_SIZE, rows * CELL_SIZE);
    container.setInteractive(hitArea, Phaser.Geom.Rectangle.Contains);
    this.input.setDraggable(container);

    return container;
}
```
- Selects random shape matrix and color from predefined arrays
- Creates a Phaser Container to hold multiple sprite blocks
- Stores matrix, color, and dimensions as container properties
- Calculates center offset for proper positioning
- Iterates through matrix, adding colored block sprites for each '1'
- Creates rectangular hit area matching shape bounds
- Makes container interactive and draggable
- Returns the constructed shape container

---

### Lines 209-228: canPlace() Method - Validation Logic
```javascript
canPlace(matrix, row, col) {
    for (let r = 0; r < matrix.length; r++) {
        for (let c = 0; c < matrix[r].length; c++) {
            if (matrix[r][c] === 1) {
                let gridR = row + r;
                let gridC = col + c;
                
                // Check bounds
                if (gridR < 0 || gridR >= GRID_SIZE || gridC < 0 || gridC >= GRID_SIZE) {
                    return false;
                }
                // Check if occupied
                if (this.gridData[gridR][gridC] !== 0) {
                    return false;
                }
            }
        }
    }
    return true;
}
```
- Takes shape matrix and target grid position
- Iterates through each cell in the shape matrix
- Calculates corresponding grid coordinates (gridR, gridC)
- Returns false if:
  - Position is outside grid bounds
  - Grid cell is already occupied (non-zero)
- Returns true if all shape cells can be placed legally

---

### Lines 230-282: placeShape() Method - Shape Placement
```javascript
placeShape(shapeObj, row, col) {
    let matrix = shapeObj.matrix;
    let color = shapeObj.color;
    let blocksPlaced = 0;

    for (let r = 0; r < matrix.length; r++) {
        for (let c = 0; c < matrix[r].length; c++) {
            if (matrix[r][c] === 1) {
                let gridR = row + r;
                let gridC = col + c;
                
                this.gridData[gridR][gridC] = 1;
                
                // Create permanent sprite on the grid
                let block = this.add.sprite(
                    GRID_X + gridC * CELL_SIZE + CELL_SIZE / 2,
                    GRID_Y + gridR * CELL_SIZE + CELL_SIZE / 2,
                    'block'
                );
                block.setTint(color);
                // Small pop animation
                block.setScale(0);
                this.tweens.add({
                    targets: block,
                    scale: 1,
                    duration: 150,
                    ease: 'Back.out'
                });
                
                this.gridBlocks[gridR][gridC] = block;
                blocksPlaced++;
            }
        }
    }

    // Update Score for placement
    this.updateScore(blocksPlaced);

    // Remove from tray
    this.trayShapes = this.trayShapes.filter(s => s !== shapeObj);
    shapeObj.destroy();

    this.checkLines();

    // Refill tray if empty
    if (this.trayShapes.length === 0) {
        this.time.delayedCall(200, () => {
            this.spawnTrayShapes();
        });
    } else {
        this.checkGameOver();
    }
}
```
- Extracts matrix and color from shape object
- Counts blocks placed for scoring
- For each '1' in matrix:
  - Updates gridData to mark cell as occupied
  - Creates visual sprite at grid position
  - Applies color tint
  - Plays "pop" scale animation (0 to 1 with Back.out easing)
  - Stores sprite reference in gridBlocks
- Updates score based on blocks placed
- Removes shape from tray and destroys the container
- Checks for completed lines
- If tray empty: waits 200ms then refills; else checks game over

---

### Lines 284-350: checkLines() Method - Line Clearing Logic
```javascript
checkLines() {
    let rowsToClear = [];
    let colsToClear = [];

    // Check Rows
    for (let r = 0; r < GRID_SIZE; r++) {
        let full = true;
        for (let c = 0; c < GRID_SIZE; c++) {
            if (this.gridData[r][c] === 0) full = false;
        }
        if (full) rowsToClear.push(r);
    }

    // Check Columns
    for (let c = 0; c < GRID_SIZE; c++) {
        let full = true;
        for (let r = 0; r < GRID_SIZE; r++) {
            if (this.gridData[r][c] === 0) full = false;
        }
        if (full) colsToClear.push(c);
    }

    let blocksToAnimate = [];

    // Gather blocks to remove
    rowsToClear.forEach(r => {
        for (let c = 0; c < GRID_SIZE; c++) {
            if (this.gridData[r][c] !== 0) {
                blocksToAnimate.push(this.gridBlocks[r][c]);
                this.gridData[r][c] = 0;
                this.gridBlocks[r][c] = null;
            }
        }
    });

    colsToClear.forEach(c => {
        for (let r = 0; r < GRID_SIZE; r++) {
            if (this.gridData[r][c] !== 0) {
                // Prevent pushing the same block twice (intersections)
                if (blocksToAnimate.indexOf(this.gridBlocks[r][c]) === -1) {
                    blocksToAnimate.push(this.gridBlocks[r][c]);
                    this.gridData[r][c] = 0;
                    this.gridBlocks[r][c] = null;
                }
            }
        }
    });

    if (blocksToAnimate.length > 0) {
        // Score for lines (combo logic can be added here)
        let linesCleared = rowsToClear.length + colsToClear.length;
        this.updateScore(linesCleared * 10);

        // Animate destruction
        this.tweens.add({
            targets: blocksToAnimate,
            scale: 0,
            alpha: 0,
            duration: 300,
            onComplete: () => {
                blocksToAnimate.forEach(b => { if(b) b.destroy(); });
                // Re-check game over as clearing lines frees up space
                this.checkGameOver(); 
            }
        });
    }
}
```
- **Phase 1 - Row Detection**: Checks each row for all occupied cells, adds complete rows to `rowsToClear`
- **Phase 2 - Column Detection**: Checks each column for all occupied cells, adds complete columns to `colsToClear`
- **Phase 3 - Block Collection**: 
  - Collects all blocks from completed rows
  - Collects blocks from completed columns (avoiding duplicates from intersections)
  - Clears gridData and gridBlocks for cleared cells
- **Phase 4 - Scoring & Animation**:
  - Awards 10 points per line cleared
  - Plays fade-out and scale-down animation over 300ms
  - On completion: destroys sprites and checks game over

---

### Lines 352-363: updateScore() Method
```javascript
updateScore(points) {
    this.score += points;
    this.scoreText.setText('SCORE: ' + this.score);
    
    // Pop effect on score
    this.tweens.add({
        targets: this.scoreText,
        scale: 1.2,
        duration: 100,
        yoyo: true
    });
}
```
- Adds points to current score
- Updates score text display
- Plays brief scale-up animation (1.2x) that yoyos back

---

### Lines 365-391: checkGameOver() Method
```javascript
checkGameOver() {
    if (this.trayShapes.length === 0) return;

    let canMove = false;

    // Brute force check: try every shape in every grid position
    for (let s = 0; s < this.trayShapes.length; s++) {
        let matrix = this.trayShapes[s].matrix;
        for (let r = 0; r < GRID_SIZE; r++) {
            for (let c = 0; c < GRID_SIZE; c++) {
                if (this.canPlace(matrix, r, c)) {
                    canMove = true;
                    break;
                }
            }
            if (canMove) break;
        }
        if (canMove) break;
    }

    if (!canMove) {
        this.triggerGameOver();
    } else {
        // If there are valid moves, make sure shapes are styled normally
        this.trayShapes.forEach(shape => shape.setAlpha(1));
    }
}
```
- Returns early if tray is empty (game continues)
- **Brute Force Algorithm**:
  - Iterates through each shape in tray
  - Tests every possible grid position (10x10 = 100 positions)
  - Uses canPlace() to check validity
- If no valid moves exist: triggers game over
- If moves exist: ensures tray shapes are fully visible (alpha = 1)

---

### Lines 393-432: triggerGameOver() Method
```javascript
triggerGameOver() {
    // Dim the remaining tray shapes to indicate they are useless
    this.trayShapes.forEach(shape => shape.setAlpha(0.3));

    // Overlay
    const overlay = this.add.graphics();
    overlay.fillStyle(0x000000, 0.7);
    overlay.fillRect(0, 0, 600, 1000);

    // Text
    this.add.text(300, 400, 'GAME OVER', {
        fontSize: '60px',
        fontFamily: 'Arial',
        fontStyle: 'bold',
        color: '#ff4757'
    }).setOrigin(0.5);

    this.add.text(300, 480, 'Final Score: ' + this.score, {
        fontSize: '30px',
        fontFamily: 'Arial',
        color: '#ffffff'
    }).setOrigin(0.5);

    // Restart Button
    let btn = this.add.graphics();
    btn.fillStyle(0x2ed573, 1);
    btn.fillRoundedRect(200, 550, 200, 60, 10);
    
    let btnText = this.add.text(300, 580, 'RESTART', {
        fontSize: '28px',
        fontFamily: 'Arial',
        fontStyle: 'bold',
        color: '#ffffff'
    }).setOrigin(0.5);

    btn.setInteractive(new Phaser.Geom.Rectangle(200, 550, 200, 60), Phaser.Geom.Rectangle.Contains);
    btn.on('pointerdown', () => {
        this.scene.restart();
    });
}
```
- Dims remaining tray shapes to 30% opacity
- Creates dark overlay (70% opacity black)
- Displays "GAME OVER" in large red text
- Shows final score below
- Creates green restart button with rounded corners
- Makes button interactive with click handler
- On click: restarts the scene (resets entire game)

---

## Phaser Configuration (Lines 435-454)

### Lines 436-449: Game Configuration Object
```javascript
const config = {
    type: Phaser.AUTO,
    // The logical size of the game (ideal for portrait mobile)
    width: 600,
    height: 1000,
    backgroundColor: '#1a1a2e',
    scale: {
        // Fit screen while preserving aspect ratio
        mode: Phaser.Scale.FIT,
        // Center the game canvas on the screen
        autoCenter: Phaser.Scale.CENTER_BOTH
    },
    scene: [GameScene]
};
```
- `type: Phaser.AUTO`: Use WebGL if available, fallback to Canvas
- `width: 600, height: 1000`: Portrait orientation, mobile-friendly dimensions
- `backgroundColor: '#1a1a2e'`: Matches body background
- `scale.mode: Phaser.Scale.FIT`: Scale to fit screen while preserving aspect ratio
- `scale.autoCenter: Phaser.Scale.CENTER_BOTH`: Center both horizontally and vertically
- `scene: [GameScene]`: Register the GameScene

### Lines 452-454: Game Initialization
```javascript
window.onload = () => {
    const game = new Phaser.Game(config);
};
```
- Waits for window load event
- Creates and starts the Phaser game instance with the config

---

## Summary

This is a complete **1010 Block Puzzle** game featuring:

| Feature | Implementation |
|---------|----------------|
| **Grid System** | 10x10 grid with logical (gridData) and visual (gridBlocks) layers |
| **Shapes** | 20 different tetromino-style shapes with random selection |
| **Drag & Drop** | Phaser's input system with offset for mobile finger visibility |
| **Placement Validation** | Bounds checking and collision detection via canPlace() |
| **Line Clearing** | Detects full rows/columns, removes blocks with animation |
| **Scoring** | Points for placement + bonus for line clears + pop animations |
| **Game Over Detection** | Brute-force algorithm checking all shapes in all positions |
| **Responsive Design** | Phaser Scale.FIT for proper mobile display |
| **No External Assets** | Dynamically generated rounded block textures |

The code demonstrates clean game development patterns including: state management separation (logic vs. visuals), reusable texture generation, tween-based animations, and event-driven architecture.