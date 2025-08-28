---
layout: default
title: Phytopolis Code Samples
description: 
---

[back to main page](./)

# PlantController (Java and LibGDX Framework)
```java
package com.syndic8.phytopolis;

import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.files.FileHandle;
import com.badlogic.gdx.utils.JsonReader;
import com.badlogic.gdx.utils.JsonValue;
import com.badlogic.gdx.utils.JsonWriter;
import com.syndic8.phytopolis.util.OSUtils;
import edu.cornell.gdiac.audio.AudioEngine;
import edu.cornell.gdiac.audio.AudioSource;
import edu.cornell.gdiac.audio.MusicQueue;
import edu.cornell.gdiac.audio.SoundEffect;

import java.text.DecimalFormat;
import java.text.DecimalFormatSymbols;
import java.util.ArrayList;
import java.util.Locale;

/**
 * Controller responsible for managing the plant growth system in the game.
 * Handles plant grid initialization, branch and leaf growth, destruction propagation,
 * and visual feedback for player interactions.
 * 
 * The plant system uses an isometric grid where players can grow branches and leaves
 * that form connected structures. When part of the plant is destroyed, the system
 * propagates destruction upward through unsupported sections.
 */
public class PlantController {
    
    // === CONSTANTS ===
    
    /** Buffer of nodes above the top of the world for collision detection */
    private static final int HEIGHT_BUFFER = 2;
    
    /** Time delay between propagations of destruction for visual effect */
    private static final float DESTRUCTION_PROPAGATION_DELAY = 0.25f;
    
    /** Buffer from top row for determining sun spawn positions */
    private static final int SUN_SPAWN_BUFFER = 4;
    
    /** Collision detection margin for world queries */
    private static final float COLLISION_MARGIN = 0.1f;
    
    /** Number of possible branch texture variants */
    private static final int BRANCH_TEXTURE_VARIANTS = 3;
    



    // === CORE SYSTEMS ===
    
    /** Queue for managing progressive destruction of plant structures */
    private final Queue<IntVector2> destructionQueue = new Queue<>(3);
    
    /** Temporary queue for safe iteration during destruction processing */
    private final Queue<IntVector2> currentDestructionQueue = new Queue<>(3);
    
    /** Temporary list for calculating next nodes during traversal */
    private final PooledList<IntVector2> nextNodes = new PooledList<>();
    
    /** Reference to the resource management system */
    private final ResourceController resourceController;
    
    /** Audio system for playing sound effects */
    private final SoundController soundController;
    
    /** Random number generator for texture variation */
    private final Random randomGenerator;
    
    /** Set for tracking removed hazards during destruction */
    private final ObjectSet<Bug> removedHazards;
    
    // === CACHED OBJECTS ===
    
    /** Reusable vector for calculations to avoid object allocation */
    private final Vector2 cacheVector = new Vector2();
    
    /** Reusable integer vector for calculations */
    private final IntVector2 cacheIntVector = new IntVector2();
    



    // === VISUAL ASSETS ===
    
    // Leaf textures
    private FilmStrip bounceAnimationTexture;
    private FilmStrip standardLeafTexture;
    private FilmStrip bouncyLeafTexture;
    private FilmStrip leafVariantOneTexture;
    private FilmStrip leafVariantTwoTexture;
    
    // Branch textures
    private FilmStrip defaultBranchTexture;
    private FilmStrip branchVariantOneTexture;
    private FilmStrip branchVariantTwoTexture;
    private FilmStrip branchVariantThreeTexture;
    private FilmStrip staticBranchTexture;
    private Texture energizedBranchTexture;
    
    /** Glow effect texture for base nodes */
    private Texture glowTexture;
    



    // === AUDIO ASSETS ===
    
    private int upgradeSoundId;
    private int destructionSoundId;
    private int leafGrowthSoundId;
    private int errorSoundId;
    



    // === GRID STATE ===
    
    /** 2D grid of plant nodes in column-major order */
    private PlantNode[][] plantGrid;
    
    /** Dimensions of the plant grid */
    private int gridWidth;
    private int gridHeight;
    
    /** Spacing between grid elements */
    private float verticalSpacing;
    private float horizontalSpacing;
    
    /** World position of grid origin */
    private float worldOriginX;
    private float worldOriginY;
    
    /** Current tilemap configuration */
    private Tilemap.TilemapParams tilemapParams;
    
    /** Coordinates of the highest point in the plant structure */
    private IntVector2 maxPlantPosition;
    
    /** Preview objects for showing potential growth locations */
    private Leaf ghostLeaf;
    private Branch ghostBranch;
    
    /** Timer for destruction propagation delays */
    private float destructionTimer = 0;
    
    /**
     * Represents the three possible directions for branch growth from any node.
     */
    public enum BranchDirection {
        LEFT, MIDDLE, RIGHT
    }
    



    // === INITIALIZATION ===
    
    /**
     * Creates a new PlantController with the specified resource controller.
     *
     * @param resourceController The resource management system for validating growth actions
     */
    public PlantController(ResourceController resourceController) {
        this.resourceController = resourceController;
        this.removedHazards = new ObjectSet<>();
        this.maxPlantPosition = new IntVector2();
        this.randomGenerator = RandomController.generator;
        this.soundController = SoundController.getInstance();
    }
    
    /**
     * Resets the plant system and initializes a new grid based on world parameters.
     * Creates an isometric grid of nodes that spans the game world.
     *
     * @param world The Box2D physics world for collision detection
     * @param params Tilemap parameters defining world dimensions and tile sizes
     */
    public void reset(World world, Tilemap.TilemapParams params) {
        this.tilemapParams = params;
        this.verticalSpacing = params.tileHeight();
        
        // Calculate grid dimensions based on world size
        this.gridWidth = Math.round(params.tilemapWidth() * (float) Math.sqrt(3));
        this.gridHeight = Math.round(params.worldHeight() / verticalSpacing) + HEIGHT_BUFFER;
        
        // Center the plant grid in the world
        float totalPlantWidth = verticalSpacing * (float) Math.sqrt(3) * (gridWidth - 1) / 2;
        this.worldOriginX = params.worldWidth() / 2 - totalPlantWidth / 2;
        this.worldOriginY = 0;
        this.horizontalSpacing = (float) Math.sqrt(3) * verticalSpacing / 2f;
        
        // Reset state
        this.removedHazards.clear();
        this.maxPlantPosition.set(-1, -1);
        
        // Initialize ghost objects for previews
        cleanupGhostObjects();
        initializeGhostObjects(params);
        
        // Create the plant grid
        this.plantGrid = new PlantNode[gridWidth][gridHeight];
        initializeGrid(world);
    }
    
    /**
     * Cleans up existing ghost objects to prevent memory leaks.
     */
    private void cleanupGhostObjects() {
        if (ghostBranch != null) {
            ghostBranch.markRemoved(true);
        }
        if (ghostLeaf != null) {
            ghostLeaf.markRemoved(true);
        }
    }
    
    /**
     * Creates new ghost objects for growth previews.
     */
    private void initializeGhostObjects(Tilemap.TilemapParams params) {
        this.ghostBranch = new Branch(0, 0, 0, Branch.BranchType.NORMAL, params, 1);
        this.ghostLeaf = new Leaf(0, 0, 1, 1, Leaf.leafType.NORMAL, params, 0.75f);
    }
    
    /**
     * Initializes the plant grid with properly spaced nodes.
     * Uses collision detection to determine which nodes should be enabled.
     *
     * @param world The Box2D world for collision queries
     */
    private void initializeGrid(World world) {
        for (int x = 0; x < gridWidth; x++) {
            for (int y = 0; y < gridHeight; y++) {
                float yOffset = (x % 2 == 1) ? verticalSpacing / 2f : 0;
                float worldX = (x * horizontalSpacing) + worldOriginX;
                float worldY = worldOriginY + yOffset + (y * verticalSpacing);
                
                boolean hasCollision = detectCollisionAt(world, worldX, worldY, yOffset == 0 && y == 0);
                
                plantGrid[x][y] = new PlantNode(
                    worldX, 
                    worldY, 
                    yOffset != 0, 
                    !hasCollision, 
                    tilemapParams
                );
            }
        }
    }
    
    /**
     * Detects collision at the specified world position.
     *
     * @param world The Box2D world to query
     * @param worldX World X coordinate
     * @param worldY World Y coordinate  
     * @param checkBelow Whether to check additional area below for base nodes
     * @return true if collision detected, false otherwise
     */
    private boolean detectCollisionAt(World world, float worldX, float worldY, boolean checkBelow) {
        final boolean[] hasCollision = {false};
        
        QueryCallback collisionCallback = new QueryCallback() {
            @Override
            public boolean reportFixture(Fixture fixture) {
                if (fixture.getBody().getUserData() instanceof Tile) {
                    hasCollision[0] = true;
                }
                return false;
            }
        };
        
        // Check primary position
        world.QueryAABB(collisionCallback,
                        worldX - COLLISION_MARGIN,
                        worldY - COLLISION_MARGIN,
                        worldX + COLLISION_MARGIN,
                        worldY + COLLISION_MARGIN);
        
        // Check below for base nodes
        if (checkBelow) {
            world.QueryAABB(collisionCallback,
                            worldX - COLLISION_MARGIN,
                            worldY + 2 * COLLISION_MARGIN,
                            worldX + COLLISION_MARGIN,
                            worldY + 4 * COLLISION_MARGIN);
        }
        
        return hasCollision[0];
    }
    



    // === GROWTH OPERATIONS ===
    
    /**
     * Attempts to grow a branch at the specified world coordinates.
     * Automatically determines the best direction based on cursor position.
     *
     * @param worldX X coordinate in world space
     * @param worldY Y coordinate in world space
     * @return The newly created branch, or null if growth failed
     */
    public Branch growBranch(float worldX, float worldY) {
        IntVector2 nodeIndex = worldToGridCoordinates(worldX, worldY);
        BranchDirection direction = determineBranchDirection(worldX, worldY);
        
        if (!validateBranchGrowth(nodeIndex.x, nodeIndex.y, direction)) {
            soundController.playSound(errorSoundId);
            return null;
        }
        
        return plantGrid[nodeIndex.x][nodeIndex.y].makeBranch(direction, Branch.BranchType.NORMAL);
    }
    
    /**
     * Validates whether a branch can be grown at the specified location and direction.
     */
    private boolean validateBranchGrowth(int gridX, int gridY, BranchDirection direction) {
        if (direction == null) {
            return false;
        }
        
        if (!resourceController.canGrowBranch()) {
            resourceController.setNotEnough(true);
            return false;
        }
        
        if (!isValidGrowthLocation(gridX, gridY, direction)) {
            return false;
        }
        
        return plantGrid[gridX][gridY].isEnabled();
    }
    
    /**
     * Checks if growth is possible in the specified direction from a node.
     */
    private boolean isValidGrowthLocation(int gridX, int gridY, BranchDirection direction) {
        if (!isWithinBounds(gridX, gridY)) {
            return false;
        }
        
        IntVector2 targetNode = getAdjacentNodePosition(gridX, gridY, direction);
        return isWithinBounds(targetNode.x, targetNode.y) && 
               isNodeEnabled(targetNode.x, targetNode.y);
    }
    
    /**
     * Attempts to grow a leaf at the specified coordinates.
     * If a leaf already exists and isn't bouncy, upgrades it to bouncy type.
     *
     * @param worldX X coordinate in world space
     * @param worldY Y coordinate in world space
     * @param leafType Type of leaf to grow
     * @param width Width of the leaf
     * @return The newly created or upgraded leaf, or null if operation failed
     */
    public Leaf growLeaf(float worldX, float worldY, Leaf.leafType leafType, float width) {
        IntVector2 nodeIndex = worldToGridCoordinates(worldX, worldY);
        int gridX = nodeIndex.x;
        int gridY = nodeIndex.y;
        
        Leaf newLeaf = null;
        
        if (hasUpgradeableLeaf(gridX, gridY)) {
            newLeaf = upgradeToBouncyLeaf(gridX, gridY, width);
        } else {
            newLeaf = createStandardLeaf(gridX, gridY, leafType, width);
        }
        
        if (newLeaf != null && newLeaf.getY() > getMaxPlantHeight()) {
            maxPlantPosition.set(gridX, gridY);
        }
        
        return newLeaf;
    }
    
    /**
     * Determines if a leaf at the specified node can be upgraded.
     */
    private boolean hasUpgradeableLeaf(int gridX, int gridY) {
        return plantGrid[gridX][gridY].hasLeaf() && 
               plantGrid[gridX][gridY].getLeafType() != Leaf.leafType.BOUNCY;
    }
    
    /**
     * Upgrades an existing leaf to bouncy type.
     */
    private Leaf upgradeToBouncyLeaf(int gridX, int gridY, float width) {
        if (!resourceController.canUpgrade()) {
            resourceController.setNotEnough(true);
            soundController.playSound(errorSoundId);
            return null;
        }
        
        plantGrid[gridX][gridY].unmakeLeaf();
        resourceController.decrementUpgrade();
        soundController.playSound(upgradeSoundId);
        
        return plantGrid[gridX][gridY].makeLeaf(Leaf.leafType.BOUNCY, width);
    }
    
    /**
     * Creates a new standard leaf at the specified location.
     */
    private Leaf createStandardLeaf(int gridX, int gridY, Leaf.leafType leafType, float width) {
        if (!validateLeafGrowth(gridX, gridY)) {
            soundController.playSound(errorSoundId);
            return null;
        }
        
        soundController.playSound(leafGrowthSoundId);
        return plantGrid[gridX][gridY].makeLeaf(leafType, width);
    }
    
    /**
     * Validates whether a leaf can be grown at the specified grid position.
     */
    private boolean validateLeafGrowth(int gridX, int gridY) {
        if (!resourceController.canGrowLeaf()) {
            resourceController.setNotEnough(true);
            return false;
        }
        
        if (!isWithinBounds(gridX, gridY)) {
            return false;
        }
        
        if (plantGrid[gridX][gridY].hasLeaf()) {
            return false;
        }
        
        // Base nodes that aren't offset cannot have leaves
        if (gridY == 0 && !plantGrid[gridX][gridY].isOffset()) {
            return false;
        }
        
        return canGrowAtGridPosition(gridX, gridY);
    }
    
    /**
     * Upgrades an existing branch to a new type.
     *
     * @param worldX X coordinate of the target node
     * @param worldY Y coordinate of the target node
     * @param direction Direction of the branch to upgrade
     * @param branchType New branch type
     * @return The upgraded branch, or null if upgrade failed
     */
    public Branch upgradeBranch(float worldX, float worldY, 
                               BranchDirection direction, Branch.BranchType branchType) {
        IntVector2 nodeIndex = worldToGridCoordinates(worldX, worldY);
        int gridX = nodeIndex.x;
        int gridY = nodeIndex.y;
        
        if (!resourceController.canUpgrade()) {
            return null;
        }
        
        resourceController.decrementUpgrade();
        soundController.playSound(upgradeSoundId);
        
        plantGrid[gridX][gridY].unmakeBranch(direction);
        return plantGrid[gridX][gridY].makeBranch(direction, branchType);
    }
    



    // === COORDINATE CONVERSION ===
    
    /**
     * Converts world coordinates to grid indices.
     *
     * @param worldX X coordinate in world space
     * @param worldY Y coordinate in world space
     * @return Grid coordinates as IntVector2
     */
    public IntVector2 worldToGridCoordinates(float worldX, float worldY) {
        int gridX = Math.round((worldX - worldOriginX) / horizontalSpacing);
        int gridY = (int) ((worldY - worldOriginY - (verticalSpacing * 0.5f * (gridX % 2))) / verticalSpacing);
        return cacheIntVector.set(gridX, gridY);
    }
    
    /**
     * Converts grid indices to world coordinates.
     *
     * @param gridX Grid X index
     * @param gridY Grid Y index
     * @return World coordinates as Vector2
     */
    public Vector2 gridToWorldCoordinates(int gridX, int gridY) {
        PlantNode node = plantGrid[gridX][gridY];
        return cacheVector.set(node.x, node.y);
    }
    
    /**
     * Determines the optimal branch direction based on cursor position relative to node center.
     *
     * @param worldX Cursor X position in world space
     * @param worldY Cursor Y position in world space
     * @return The best branch direction, or null if position is invalid
     */
    public BranchDirection determineBranchDirection(float worldX, float worldY) {
        IntVector2 nodeIndex = worldToGridCoordinates(worldX, worldY);
        int gridX = nodeIndex.x;
        int gridY = nodeIndex.y;
        
        if (!isNodeEnabled(gridX, gridY)) {
            return null;
        }
        
        Vector2 nodeCenter = gridToWorldCoordinates(gridX, gridY);
        float angle = (float) Math.atan2(worldY - nodeCenter.y, worldX - nodeCenter.x);
        
        // Normalize to 0-2π range
        angle = (angle + (float) (2 * Math.PI)) % (float) (2 * Math.PI);
        
        // Only allow upward growth
        if (angle >= Math.PI) {
            return null;
        }
        
        BranchDirection direction = angleToDirection(angle);
        
        if (branchExists(gridX, gridY, direction) || !canGrowAtGridPosition(gridX, gridY)) {
            return null;
        }
        
        return direction;
    }
    
    /**
     * Converts an angle to the corresponding branch direction.
     */
    private BranchDirection angleToDirection(float angle) {
        if (angle < Math.PI / 3) {
            return BranchDirection.RIGHT;
        } else if (angle < 2 * Math.PI / 3) {
            return BranchDirection.MIDDLE;
        } else {
            return BranchDirection.LEFT;
        }
    }
    



    // === DESTRUCTION SYSTEM ===
    
    /**
     * Processes queued destruction events with timing delays for visual effect.
     * Should be called every frame to handle progressive plant destruction.
     *
     * @param deltaTime Time elapsed since last frame
     * @return Set of bugs that were removed due to leaf destruction
     */
    public ObjectSet<Bug> propagateDestruction(float deltaTime) {
        destructionTimer -= deltaTime;
        removedHazards.clear();
        
        if (destructionQueue.isEmpty() || destructionTimer > 0) {
            return removedHazards;
        }
        
        processNextDestructionBatch();
        return removedHazards;
    }
    
    /**
     * Processes the next batch of nodes in the destruction queue.
     */
    private void processNextDestructionBatch() {
        currentDestructionQueue.clear();
        destructionQueue.forEach(position -> currentDestructionQueue.addLast(position));
        
        for (IntVector2 position : currentDestructionQueue) {
            destroyNodeRecursively(position.x, position.y);
            destructionQueue.removeValue(position, true);
        }
    }
    
    /**
     * Recursively destroys plant structures starting from the specified node.
     * Queues additional destruction for any newly unsupported sections.
     *
     * @param gridX Grid X index of the destruction starting point
     * @param gridY Grid Y index of the destruction starting point
     */
    private void destroyNodeRecursively(int gridX, int gridY) {
        if (!isWithinBounds(gridX, gridY) || isNodeEmpty(gridX, gridY)) {
            return;
        }
        
        PlantNode node = plantGrid[gridX][gridY];
        
        // Handle hazard removal
        if (node.getHazard() instanceof Bug bug) {
            removedHazards.add(bug);
        }
        
        // Remove node contents
        node.unmakeLeaf();
        node.removeHazard();
        
        // Process all connected branches
        for (BranchDirection direction : BranchDirection.values()) {
            if (node.hasBranchInDirection(direction)) {
                processBranchDestruction(gridX, gridY, direction);
            }
        }
        
        soundController.playSound(destructionSoundId);
    }
    
    /**
     * Handles the destruction of a specific branch and queues connected nodes if needed.
     */
    private void processBranchDestruction(int gridX, int gridY, BranchDirection direction) {
        plantGrid[gridX][gridY].unmakeBranch(direction);
        IntVector2 connectedNode = getAdjacentNodePosition(gridX, gridY, direction);
        
        if (!canGrowAtGridPosition(connectedNode.x, connectedNode.y)) {
            queueDestruction(connectedNode.x, connectedNode.y);
        }
    }
    
    /**
     * Schedules a node for destruction by adding it to the queue.
     *
     * @param gridX Grid X index of the node to destroy
     * @param gridY Grid Y index of the node to destroy
     */
    public void queueDestruction(int gridX, int gridY) {
        destructionQueue.addFirst(new IntVector2(gridX, gridY));
        destructionTimer = DESTRUCTION_PROPAGATION_DELAY;
    }
    



    // === VALIDATION AND QUERIES ===
    
    /**
     * Determines if growth is possible at the specified grid position.
     * A position is growable if it's connected to the plant structure below.
     *
     * @param gridX Grid X index
     * @param gridY Grid Y index
     * @return true if growth is possible at this position
     */
    public boolean canGrowAtGridPosition(int gridX, int gridY) {
        // Base nodes are always growable
        boolean isBaseNode = (gridX % 2 == 0);
        if (gridY == 0 && isBaseNode) {
            return true;
        }
        
        if (!isWithinBounds(gridX, gridY) || !plantGrid[gridX][gridY].isEnabled()) {
            return false;
        }
        
        // Check for supporting connections from below
        int offsetAdjustment = isBaseNode ? 1 : 0;
        
        boolean hasDirectSupport = isWithinBounds(gridX, gridY - 1) && 
                                  plantGrid[gridX][gridY - 1].hasBranchInDirection(BranchDirection.MIDDLE);
        
        boolean hasLeftSupport = isWithinBounds(gridX - 1, gridY - offsetAdjustment) &&
                                plantGrid[gridX - 1][gridY - offsetAdjustment].hasBranchInDirection(BranchDirection.RIGHT);
        
        boolean hasRightSupport = isWithinBounds(gridX + 1, gridY - offsetAdjustment) &&
                                 plantGrid[gridX + 1][gridY - offsetAdjustment].hasBranchInDirection(BranchDirection.LEFT);
        
        return hasDirectSupport || hasLeftSupport || hasRightSupport;
    }
    
    /**
     * Checks if the specified world position is suitable for growth.
     */
    public boolean canGrowAt(float worldX, float worldY) {
        IntVector2 nodeIndex = worldToGridCoordinates(worldX, worldY);
        return canGrowAtGridPosition(nodeIndex.x, nodeIndex.y);
    }
    
    /**
     * Verifies that grid coordinates are within valid bounds.
     */
    public boolean isWithinBounds(int gridX, int gridY) {
        return gridX >= 0 && gridY >= 0 && gridX < gridWidth && gridY < gridHeight;
    }
    
    /**
     * Checks if a node is enabled (not blocked by terrain).
     */
    public boolean isNodeEnabled(int gridX, int gridY) {
        return isWithinBounds(gridX, gridY) && plantGrid[gridX][gridY].isEnabled();
    }
    
    /**
     * Checks if a node has no branches or leaves.
     */
    public boolean isNodeEmpty(int gridX, int gridY) {
        return isWithinBounds(gridX, gridY) && plantGrid[gridX][gridY].isEmpty();
    }
    
    /**
     * Checks if a branch exists at the specified location and direction.
     */
    public boolean branchExists(int gridX, int gridY, BranchDirection direction) {
        return isWithinBounds(gridX, gridY) && 
               plantGrid[gridX][gridY].hasBranchInDirection(direction);
    }
    
    /**
     * Checks if a leaf exists at the specified world position.
     */
    public boolean hasLeaf(Vector2 position) {
        IntVector2 nodeIndex = worldToGridCoordinates(position.x, position.y);
        return hasLeaf(nodeIndex.x, nodeIndex.y);
    }
    
    /**
     * Checks if a leaf exists at the specified grid position.
     */
    public boolean hasLeaf(int gridX, int gridY) {
        return isWithinBounds(gridX, gridY) && plantGrid[gridX][gridY].hasLeaf();
    }
    



    // === HAZARD MANAGEMENT ===
    
    /**
     * Places a hazard at the specified grid position.
     */
    public void setHazard(int gridX, int gridY, Hazard hazard) {
        if (isWithinBounds(gridX, gridY)) {
            plantGrid[gridX][gridY].setHazard(hazard);
        }
    }
    
    /**
     * Removes a hazard from the specified grid position.
     */
    public void removeHazard(int gridX, int gridY) {
        if (isWithinBounds(gridX, gridY)) {
            plantGrid[gridX][gridY].removeHazard();
        }
    }
    
    /**
     * Checks if a hazard exists at the specified grid position.
     */
    public boolean hasHazard(int gridX, int gridY) {
        return isWithinBounds(gridX, gridY) && plantGrid[gridX][gridY].hasHazard();
    }
    
    /**
     * Removes a specific hazard from all nodes where it's present.
     */
    public void removeHazardFromAllNodes(Hazard hazard) {
        for (int x = 0; x < gridWidth; x++) {
            for (int y = 0; y < gridHeight; y++) {
                if (plantGrid[x][y].getHazard() == hazard) {
                    plantGrid[x][y].removeHazard();
                    
                    if (plantGrid[x][y].hasLeaf() && plantGrid[x][y].getLeaf().fullyEaten()) {
                        plantGrid[x][y].unmakeLeaf();
                    }
                    return;
                }
            }
        }
    }
    
    /**
     * Removes bugs from fully eaten leaves and returns the removed bugs.
     *
     * @return Set of bugs that were removed from dead leaves
     */
    public ObjectSet<Bug> removeDeadLeafBugs() {
        removedHazards.clear();
        
        for (int x = 0; x < gridWidth; x++) {
            for (int y = 0; y < gridHeight; y++) {
                if (isLeafFullyEaten(x, y)) {
                    processDeadLeaf(x, y);
                }
            }
        }
        
        return removedHazards;
    }
    
    /**
     * Checks if a leaf at the specified position is fully eaten.
     */
    private boolean isLeafFullyEaten(int gridX, int gridY) {
        return plantGrid[gridX][gridY].hasLeaf() && 
               plantGrid[gridX][gridY].getLeaf().fullyEaten();
    }
    
    /**
     * Processes a dead leaf by removing associated bugs and the leaf itself.
     */
    private void processDeadLeaf(int gridX, int gridY) {
        Hazard hazard = plantGrid[gridX][gridY].getHazard();
        if (hazard instanceof Bug bug) {
            removedHazards.add(bug);
            plantGrid[gridX][gridY].removeHazard();
            plantGrid[gridX][gridY].unmakeLeaf();
        }
    }



    
    // === UTILITY METHODS ===
    
    /**
     * Calculates the adjacent node position in the specified direction.
     */
    private IntVector2 getAdjacentNodePosition(int gridX, int gridY, BranchDirection direction) {
        int offsetAdjustment = plantGrid[gridX][gridY].isOffset() ? 1 : 0;
        
        switch (direction) {
            case LEFT:
                return cacheIntVector.set(gridX - 1, gridY + offsetAdjustment);
            case RIGHT:
                return cacheIntVector.set(gridX + 1, gridY + offsetAdjustment);
            case MIDDLE:
                return cacheIntVector.set(gridX, gridY + 1);
            default:
                return null;
        }
    }
    
    /**
     * Selects a random branch texture for visual variety.
     */
    private FilmStrip getRandomBranchTexture() {
        int textureIndex = randomGenerator.nextInt(BRANCH_TEXTURE_VARIANTS);
        
        switch (textureIndex) {
            case 0: return branchVariantOneTexture;
            case 1: return branchVariantTwoTexture;
            case 2: return branchVariantThreeTexture;
            default: return branchVariantOneTexture;
        }
    }
    



    // === PLANT HEIGHT TRACKING ===
    
    /**
     * Returns the Y coordinate of the highest point in the plant structure.
     */
    public float getMaxPlantHeight() {
        if (maxPlantPosition.x != -1 && 
            isWithinBounds(maxPlantPosition.x, maxPlantPosition.y)) {
            return plantGrid[maxPlantPosition.x][maxPlantPosition.y].y;
        }
        return 0;
    }
    
    /**
     * Recalculates the maximum plant height after destruction operations.
     * This is an expensive operation and should only be called when necessary.
     */
    public void recalculateMaxPlantPosition() {
        IntVector2 highestPosition = new IntVector2(0, 0);
        
        for (int x = 0; x < gridWidth; x++) {
            IntVector2 columnHighest = findHighestPositionFromNode(x, 0);
            if (getNodeHeight(columnHighest) > getNodeHeight(highestPosition)) {
                highestPosition.set(columnHighest);
            }
        }
        
        maxPlantPosition.set(highestPosition);
    }
    
    /**
     * Recursively finds the highest reachable position from a starting node.
     */
    private IntVector2 findHighestPositionFromNode(int gridX, int gridY) {
        PlantNode node = plantGrid[gridX][gridY];
        nextNodes.clear();
        
        // Explore all connected branches
        for (BranchDirection direction : BranchDirection.values()) {
            if (node.hasBranchInDirection(direction)) {
                IntVector2 connectedNode = getAdjacentNodePosition(gridX, gridY, direction);
                IntVector2 highestFromBranch = findHighestPositionFromNode(connectedNode.x, connectedNode.y);
                nextNodes.push(new IntVector2(highestFromBranch));
            }
        }
        
        // Find the highest position among all connected branches
        IntVector2 highestPosition = new IntVector2(gridX, gridY);
        for (IntVector2 candidate : nextNodes) {
            if (getNodeHeight(candidate) > getNodeHeight(highestPosition)) {
                highestPosition.set(candidate);
            }
        }
        
        nextNodes.clear();
        return highestPosition;
    }
    
    /**
     * Gets the world Y coordinate of a node at the specified grid position.
     */
    private float getNodeHeight(IntVector2 position) {
        return plantGrid[position.x][position.y].y;
    }



    
    // === GAME MECHANICS ===
    
    /**
     * Counts timer deductions caused by bugs eating leaves.
     * Used for gameplay mechanics where damaged leaves affect game timer.
     */
    public int countTimerDeductions() {
        int deductions = 0;
        
        for (int x = 0; x < gridWidth; x++) {
            for (int y = 0; y < gridHeight; y++) {
                Leaf leaf = plantGrid[x][y].getLeaf();
                if (leaf != null && leaf.healthBelowMark()) {
                    deductions++;
                }
            }
        }
        
        return deductions;
    }
    
    /**
     * Returns valid X coordinates for sun spawning near the top of the plant structure.
     * Used by the sun controller to determine spawn positions.
     */
    public PooledList<Float> getValidSunSpawnPositions() {
        PooledList<Float> spawnPositions = new PooledList<>();
        int spawnRow = gridHeight - 1 - SUN_SPAWN_BUFFER;
        
        for (int x = 0; x < gridWidth; x++) {
            if (plantGrid[x][spawnRow].isEnabled()) {
                spawnPositions.push(plantGrid[x][0].x);
            }
        }
        
        return spawnPositions;
    }



    
    // === VISUAL FEEDBACK ===
    
    /**
     * Renders a ghost branch preview at the optimal growth position.
     * Shows players where a branch would be placed if they clicked.
     */
    public void drawGhostBranch(GameCanvas canvas, float worldX, float worldY) {
        IntVector2 nodeIndex = worldToGridCoordinates(worldX, worldY);
        int gridX = nodeIndex.x;
        int gridY = nodeIndex.y;
        BranchDirection direction = determineBranchDirection(worldX, worldY);
        
        if (!shouldDrawGhostBranch(gridX, gridY, direction)) {
            return;
        }
        
        configureGhostBranch(gridX, gridY, direction);
        ghostBranch.drawGhost(canvas);
    }
    
    /**
     * Validates whether a ghost branch should be drawn.
     */
    private boolean shouldDrawGhostBranch(int gridX, int gridY, BranchDirection direction) {
        return isWithinBounds(gridX, gridY) &&
               direction != null &&
               !plantGrid[gridX][gridY].hasBranchInDirection(direction) &&
               isValidGrowthLocation(gridX, gridY, direction) &&
               canGrowAtGridPosition(gridX, gridY) &&
               resourceController.canGrowBranch();
    }
    
    /**
     * Configures the ghost branch for rendering.
     */
    private void configureGhostBranch(int gridX, int gridY, BranchDirection direction) {
        PlantNode node = plantGrid[gridX][gridY];
        ghostBranch.setX(node.getX());
        ghostBranch.setY(node.getY());
        ghostBranch.setAngle(getBranchAngle(direction));
        ghostBranch.setFilmStrip(defaultBranchTexture);
    }
    
    /**
     * Returns the angle for a branch in the specified direction.
     */
    private float getBranchAngle(BranchDirection direction) {
        switch (direction) {
            case MIDDLE: return 0;
            case LEFT: return (float) Math.PI / 3;
            case RIGHT: return (float) -Math.PI / 3;
            default: return 0;
        }
    }
    
    /**
     * Renders a ghost leaf preview at the optimal growth position.
     * This is used for when the user hovers their cursor over a node, 
     * but has yet to place a leaf
     */
    public void drawGhostLeaf(GameCanvas canvas, Leaf.leafType leafType, 
                             float leafWidth, float worldX, float worldY) {
        IntVector2 nodeIndex = worldToGridCoordinates(worldX, worldY);
        int gridX = nodeIndex.x;
        int gridY = nodeIndex.y;
        
        if (!shouldDrawGhostLeaf(gridX, gridY)) {
            return;
        }
        
        configureGhostLeaf(gridX, gridY, leafType, leafWidth);
        ghostLeaf.drawGhost(canvas);
    }
    
    /**
     * Validates whether a ghost leaf should be drawn.
     */
    private boolean shouldDrawGhostLeaf(int gridX, int gridY) {
        return isWithinBounds(gridX, gridY) &&
               !plantGrid[gridX][gridY].hasLeaf() &&
               !(gridY <= 0 && !plantGrid[gridX][gridY].isOffset()) &&
               canGrowAtGridPosition(gridX, gridY) &&
               resourceController.canGrowLeaf();
    }
    
    /**
     * Configures the ghost leaf for rendering.
     */
    private void configureGhostLeaf(int gridX, int gridY, Leaf.leafType leafType, float leafWidth) {
        PlantNode node = plantGrid[gridX][gridY];
        ghostLeaf.setPosition(node.getX(), node.getY());
        ghostLeaf.setDimension(leafWidth, PlantNode.LEAF_HEIGHT);
        ghostLeaf.setType(leafType);
        ghostLeaf.setFilmStrip(getLeafTexture(leafType));
    }
    
    /**
     * Returns the appropriate texture for the specified leaf type.
     */
    private FilmStrip getLeafTexture(Leaf.leafType leafType) {
        switch (leafType) {
            case BOUNCY: return bouncyLeafTexture;
            case NORMAL1: return leafVariantOneTexture;
            case NORMAL2: return leafVariantTwoTexture;
            case NORMAL:
            default: return standardLeafTexture;
        }
    }
    
    /**
     * Renders glow effects at enabled base nodes to guide player placement.
     */
    public void drawGlow(GameCanvas canvas) {
        float scaleX = tilemapParams.tileWidth() / glowTexture.getWidth();
        float scaleY = tilemapParams.tileHeight() / glowTexture.getHeight();
        
        for (PlantNode[] column : plantGrid) {
            PlantNode baseNode = column[0];
            if (!baseNode.isOffset() && baseNode.isEnabled()) {
                canvas.draw(glowTexture,
                           Color.WHITE,
                           glowTexture.getWidth() / 2f,
                           0,
                           baseNode.x,
                           baseNode.y,
                           0,
                           scaleX,
                           scaleY);
            }
        }
    }
    



    // === GETTERS AND INFORMATION ===
    
    public int getGridWidth() { return gridWidth; }
    public int getGridHeight() { return gridHeight; }
    public int getMaxPlantXIndex() { return maxPlantPosition.x; }
    public int getMaxPlantYIndex() { return maxPlantPosition.y; }
    public ResourceController getResourceController() { return resourceController; }
    
    /**
     * Gets the branch type at the specified location and direction.
     */
    public Branch.BranchType getBranchType(int gridX, int gridY, BranchDirection direction) {
        return isWithinBounds(gridX, gridY) ? 
               plantGrid[gridX][gridY].getBranchType(direction) : null;
    }
    
    /**
     * Gets the leaf type at the specified grid position.
     */
    public Leaf.leafType getLeafType(int gridX, int gridY) {
        return isWithinBounds(gridX, gridY) ? 
               plantGrid[gridX][gridY].getLeafType() : null;
    }
    
    /**
     * Checks if the specified column uses offset positioning.
     */
    public boolean isColumnOffset(int columnIndex) {
        return columnIndex < gridWidth && plantGrid[columnIndex][0].isOffset();
    }
    


    
    // === ASSET MANAGEMENT ===
    
    /**
     * Loads all visual and audio assets required by the plant system.
     * Should be called during game initialization.
     *
     * @param directory Asset directory containing game resources
     */
    public void loadAssets(AssetDirectory directory) {
        loadBranchTextures(directory);
        loadLeafTextures(directory);
        loadSpecialTextures(directory);
        loadSoundEffects(directory);
    }
    
    /**
     * Loads all branch-related textures.
     */
    private void loadBranchTextures(AssetDirectory directory) {
        defaultBranchTexture = createFilmStrip(directory, "gameplay:branch", 1, 5, 5);
        branchVariantOneTexture = createFilmStrip(directory, "gameplay:branch1", 1, 5, 5);
        branchVariantTwoTexture = createFilmStrip(directory, "gameplay:branch2", 1, 5, 5);
        branchVariantThreeTexture = createFilmStrip(directory, "gameplay:branch3", 1, 5, 5);
        staticBranchTexture = createFilmStrip(directory, "gameplay:branch", 1, 5, 5);
        staticBranchTexture.setFrame(4);
    }
    
    /**
     * Loads all leaf-related textures.
     */
    private void loadLeafTextures(AssetDirectory directory) {
        standardLeafTexture = createFilmStrip(directory, "gameplay:leaf", 1, 9, 9);
        leafVariantOneTexture = createFilmStrip(directory, "gameplay:leaf1", 1, 9, 9);
        leafVariantTwoTexture = createFilmStrip(directory, "gameplay:leaf2", 1, 9, 9);
        bouncyLeafTexture = createFilmStrip(directory, "gameplay:bouncy", 1, 7);
        bounceAnimationTexture = createFilmStrip(directory, "gameplay:bouncy_bounce", 1, 6);
    }
    
    /**
     * Loads special effect textures.
     */
    private void loadSpecialTextures(AssetDirectory directory) {
        energizedBranchTexture = directory.getEntry("gameplay:enbranch", Texture.class);
        glowTexture = directory.getEntry("gameplay:glow", Texture.class);
    }
    
    /**
     * Loads sound effects and registers them with the sound controller.
     */
    private void loadSoundEffects(AssetDirectory directory) {
        upgradeSoundId = soundController.addSoundEffect(
            directory.getEntry("upgradeleaf", SoundEffect.class));
        destructionSoundId = soundController.addSoundEffect(
            directory.getEntry("destroyplant2", SoundEffect.class));
        leafGrowthSoundId = soundController.addSoundEffect(
            directory.getEntry("growleaf", SoundEffect.class));
        errorSoundId = soundController.addSoundEffect(
            directory.getEntry("errorsound", SoundEffect.class));
    }
    
    /**
     * Helper method to create FilmStrip objects with consistent error handling.
     */
    private FilmStrip createFilmStrip(AssetDirectory directory, String assetName, 
                                     int rows, int cols, int frameCount) {
        Texture texture = directory.getEntry(assetName, Texture.class);
        return new FilmStrip(texture, rows, cols, frameCount);
    }
    
    /**
     * Helper method to create FilmStrip objects with default frame count.
     */
    private FilmStrip createFilmStrip(AssetDirectory directory, String assetName, 
                                     int rows, int cols) {
        return createFilmStrip(directory, assetName, rows, cols, cols);
    }
}
```

[back to main page](./)