# Svelte + TS + Vite

- The project uses pnpm.

run:

```shell
 pnpm install 
 ```

followed by:

```shell
pnpm run dev
```

to execute the dev environment.

# Map layer
 
## Map navigation controls with zooming, moving and rotating the base map Interaction layer with following capabilities:
        
### Add object of type SQUARE
 - A SQUARE object starts its life in the territory of Estonia
 - It moves with a random speed between 50 and 80 km/h
- It moves on a straight path on a Great Circle and, given sufficient time, returns to the starting position and keeps going
        
 ### Add object of type CIRCLE
- A CIRCLE object starts its life in the territory of Estonia
-   It moves with a random speed between 110 and 300 km/h
- It moves on a circular path with a radius of 10000 - 30000m and after it has returned to origin point, dissapears

### Add object of type TRIANGLE
- A TRIANGLE object starts its life in the territory of Estonia
- It moves with a random speed between 1700 and 2200 km/h
- Its destination is a point randomly chosen anywere on Earth
- It moves on a shortest path from origin to destination point. Bear in mind that the shortest path is not a straight line.
- The element has a lifespan of one hour or ends its life upon arrival to destination
    Object control layer with capabilities to display and hide any of the object types of SQUARE, CIRCLE and TRIANGLE
   
### Each individeual object of type SQUARE, CIRCLE and TRIANGLE are clickable

On activating, display:
- speed
- current location
- time to expire
- current trajectory