<template>
  <div>
    <h1>{{ msg }}</h1>
    <div>
      <input
        type="number"
        min="1"
        max="100"
        v-model="numColumns"
        @change="regenerateCells"
      />
      x
      <input
        type="number"
        min="1"
        max="100"
        v-model="numRows"
        @change="regenerateCells"
      />
    </div>
    <div class="swatches-container">
      <div
        :class="[
          'swatch-container',
          selectedSwatchIndex == index
            ? 'swatch-selected'
            : 'swatch-unselected',
        ]"
        v-for="(swatch, index) in swatches"
        :key="index"
        :value="index"
      >
        <button @click="selectedSwatchIndex = index">
          Swatch {{ index + 1 }}
        </button>

        <br />
        <input type="color" v-model="swatch.color" />
        <br />
        {{ swatchCounts[index] }}
      </div>
      <div class="swatch-container">
        <button @click="addSwatch">+</button>
      </div>
    </div>
    <div style="float: right">
      <button @click="load">Load</button>
      <button @click="save">Save</button>
    </div>
    <div>Brush Size: <input type="number" v-model="brushSize" /></div>
    <div class="quilt" @mousedown="onMouseDown" @mouseup="onMouseUp">
      <div v-for="(row, rowIndex) in rows" :key="rowIndex" class="quilt-row">
        <span
          v-for="(cell, cellIndex) in row"
          :key="cellIndex"
          class="quilt-cell"
          @mouseover="onMouseOver(cell)"
          @mousedown="(ev) => onMouseDownOnCell(cell, ev)"
          :style="{ backgroundColor: swatches[cell.swatchIndex].color }"
          >{{ cell.name }}</span
        >
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Options, Vue } from "vue-class-component";

type Swatch = {
  color: string;
};

type Cell = {
  name: string;
  swatchIndex: number;
  row: number;
  col: number;
};

@Options({
  props: {
    msg: String,
  },
})
export default class HelloWorld extends Vue {
  msg!: string;

  brushSize = 1;
  numRows = 40;
  numColumns = 64;
  selectedSwatchIndex = 0;

  rows: Cell[][] = [];
  swatches: Swatch[] = [
    { color: "#FFAAAA" },
    { color: "#AAFFAA" },
    { color: "#AAAAFF" },
  ];

  swatchCounts: number[] = [];

  created() {
    this.regenerateCells();
    this.load();
  }

  addRow() {
    this.numRows++;
    this.regenerateCells();
  }

  removeRow() {
    this.numRows--;
    this.regenerateCells();
  }

  addColumn() {
    this.numColumns++;
    this.regenerateCells();
  }

  removeColumn() {
    this.numColumns--;
    this.regenerateCells();
  }

  regenerateCells() {
    var oldRows = this.rows;
    this.rows = [];
    for (var i = 0; i < this.numRows; i++) {
      this.rows[i] = [];
      for (var j = 0; j < this.numColumns; j++) {
        this.rows[i][j] = { name: `${i},${j}`, swatchIndex: 0, row: i, col: j };
        if (oldRows[i] && oldRows[i][j])
          this.rows[i][j].swatchIndex = oldRows[i][j].swatchIndex;
      }
    }
    this.queueSave();
  }

  isMouseDown = false;

  onMouseDownOnCell(cell: Cell, event: MouseEvent) {
    if (event.button !== 0) return;
    this.isMouseDown = true;
    this.setColor(cell);
  }

  onMouseDown(event: MouseEvent) {
    if (event.button !== 0) return;
    this.isMouseDown = true;
  }

  onMouseUp() {
    this.isMouseDown = false;
  }

  onMouseOver(cell: Cell) {
    if (this.isMouseDown) {
      this.setColor(cell);
    }
  }

  setColor(cell: Cell) {
    var leftIndex = cell.col - Math.floor((this.brushSize - 1) / 2);
    var rightIndex = cell.col + Math.ceil((this.brushSize - 1) / 2);
    var topIndex = cell.row - Math.floor((this.brushSize - 1) / 2);
    var bottomIndex = cell.row + Math.ceil((this.brushSize - 1) / 2);

    if (leftIndex < 0) leftIndex = 0;
    if (rightIndex >= this.numColumns) rightIndex = this.numColumns - 1;
    if (topIndex < 0) topIndex = 0;
    if (bottomIndex >= this.numRows) bottomIndex = this.numRows - 1;

    for (var colIndex = leftIndex; colIndex <= rightIndex; colIndex++) {
      for (var rowIndex = topIndex; rowIndex <= bottomIndex; rowIndex++) {
        var cellToChange = this.rows[rowIndex][colIndex];
        if (cellToChange.swatchIndex !== this.selectedSwatchIndex) {
          cellToChange.swatchIndex = this.selectedSwatchIndex;
        }
      }
    }
    this.queueSave();
  }

  addSwatch() {
    this.swatches = [...this.swatches, { color: "#FFFFFF" }];
    this.selectedSwatchIndex = this.swatches.length - 1;
    this.queueSave();
  }

  countSwatches() {
    this.swatchCounts = this.swatches.map(() => 0);
    this.rows.forEach((row) =>
      row.forEach((cell) => this.swatchCounts[cell.swatchIndex]++)
    );
  }

  saveTimeout = 0;
  queueSave() {
    if (!this.saveTimeout) {
      this.saveTimeout = setTimeout(() => {
        this.save();
        this.saveTimeout = 0;
      }, 2000);
    }
  }

  save() {
    const model = {
      grid: this.rows,
      swatches: this.swatches,
      selectedSwatchIndex: this.selectedSwatchIndex,
      brushSize: this.brushSize,
    };

    localStorage.setItem("quilt", JSON.stringify(model));
    this.countSwatches();
  }

  load() {
    const json = localStorage.getItem("quilt");
    if (json) {
      const model = JSON.parse(json);
      this.rows = model.grid;
      this.swatches = model.swatches;
      this.selectedSwatchIndex = model.selectedSwatchIndex || 0;
      this.brushSize = model.brushSize || 1;

      this.numRows = this.rows.length;
      this.numColumns = this.rows[0].length;

      this.regenerateCells();
      this.countSwatches();
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.quilt {
  --cell-size: 18px;
  --label-size: 8px;
  background: black;
  padding: 1em;
  overflow: auto;
  display: block;
  user-select: none;
}

.quilt-row {
  white-space: nowrap;
  line-height: var(--cell-size);
  margin-bottom: -2px;
}

.quilt-cell {
  background: #ddddff;
  display: inline-block;
  width: var(--cell-size);
  height: var(--cell-size);
  line-height: var(--cell-size);
  text-align: center;
  margin: 1px;
  font-size: var(--label-size);
}

.swatches-container {
  display: flex;
  flex-direction: row;
}
.swatch-container {
  border: 2px solid transparent;
  padding: 2px;
}
.swatch-selected {
  background: pink;
  border: 2px solid black;
}
</style>
