# Display Grid Gotchas
Display grid is incredibly powerful, but can also be incredibly confusing with the millions of ways to configure named areas, lines, and more.

## Grid Template Area Syntax
While naming the template areas in the grid parent, the names are wrapped in quotations, however when referencing the area name from the child *do not* use quotations
```css
parent {
  grid-template-areas: 'area-1 area-2 area-3';
}

child {
  /* Will NOT work */
  grid-area: 'area-1';
  /* Proper syntax */
  grid-area: area-1;
}
```

## Implicit and Explicit Grids
There are two types of grid cells in a display grid element, [explicit and implicit](https://css-tricks.com/difference-explicit-implicit-grids).

### Explicit Cells
These are the cells that you've defined yourself explicitly using
```css
grid-template-rows: x x x;
grid-template-columns: x x x;
```
This would define a 3x3 grid, with set cell dimensions, however you can still place items outside of the 3x3 grid (ie. `grid-row-start: 4;`) which is where Implicit cells come in

### Implicit Cells
These are all cells not defined by a template in the grid and are assigned automatically, however you can still style these implicit cells using
```css
grid-auto-rows: x;
grid-auto-columns: x;
```
that will apply to all items that appear outside of the explicit grid.
