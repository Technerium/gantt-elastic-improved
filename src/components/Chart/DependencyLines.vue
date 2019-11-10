<!--
/**
 * @fileoverview DependencyLines component
 * @license MIT
 * @author Rafal Pospiech <neuronet.io@gmail.com>
 * @package GanttElastic
 */
-->
<template>
  <svg
    x="0"
    y="0"
    width="100%"
    height="100%"
    class="gantt-elastic__chart-dependency-lines-container"
    :style="{ ...root.style['chart-dependency-lines-container'] }"
  >
    <defs>
      <marker id="markerCircle" markerWidth="8" markerHeight="8" refX="5" refY="5">
        <circle cx="5" cy="5" r="1" style="stroke: none; fill:#000000;"/>
      </marker>

      <marker id="markerArrow" markerWidth="9" markerHeight="9" refX="2" refY="4.5"
              orient="auto">
        <path d="M2,2 L2,7 L6,4.5 L2,2" style="fill: #000000;" />
      </marker>
    </defs>
    <g v-for="task in dependencyTasks" :key="task.id" :task="task">
      <path
        class="gantt-elastic__chart-dependency-lines-path gl-hover"
        :style="{...{stroke: dependencyLine.lineColor}, ...root.style['chart-dependency-lines-path'], ...task.style['chart-dependency-lines-path'] }"
        v-for="dependencyLine in task.dependencyLines"
        :key="dependencyLine.id"
        :task="task"
        :d="dependencyLine.points"
      ></path>
    </g>
  </svg>
</template>

<script>
export default {
  name: 'DependencyLines',
  inject: ['root'],
  props: ['tasks'],
  data() {
    return {};
  },
  methods: {
    getOffset(){
      return 15;
    },
    turnVert(startX, startY, roundingRadius, directionX, directionY){
      let stopX = startX + roundingRadius*directionX;
      let stopY = startY + roundingRadius*directionY;
      let cmd = `
            Q ${stopX},${startY} ${stopX},${stopY}
      `;
      return {
        x: stopX,
        y: stopY,
        cmd: cmd
      }
    },
    turnHor(startX, startY, roundingRadius, directionX, directionY){
      let stopX = startX + roundingRadius*directionX;
      let stopY = startY + roundingRadius*directionY;
      let cmd = `
            Q ${startX},${stopY} ${stopX},${stopY}
      `;
      return {
        x: stopX,
        y: stopY,
        cmd: cmd
      }
    },

    getHLineCommand(startX, startY, stopX, stopY, roundingRadius){
      let directionX = 1;
      let directionY = 1;
      let distanceX = stopX - startX;
      let distanceY = stopY - startY;
      if(distanceX < 0){
        directionX = -1;
      }
      if(distanceY < 0){
        directionY = -1;
      }

      let command = `
            L ${startX + distanceX/2 - roundingRadius * directionX},${startY}
            Q ${startX + distanceX/2},${startY} ${startX + distanceX/2},${startY + roundingRadius*directionY}
            L ${startX + distanceX/2},${stopY - roundingRadius*directionY}
            Q ${startX + distanceX/2},${stopY} ${startX + distanceX/2 + roundingRadius * directionX},${stopY}
            L ${stopX},${stopY}
      `;

      return command
    },
    /**
     * Get path points
     *
     * @param {any} fromTaskId
     * @param {any} toTaskId
     * @returns {string}
     */
    getPointsStart2Start(fromTaskId, toTaskId) {
      const fromTask = this.root.getTask(fromTaskId);
      const toTask = this.root.getTask(toTaskId);
      if (
        fromTask === null ||
        toTask === null ||
        !this.root.isTaskVisible(toTask) ||
        !this.root.isTaskVisible(fromTask)
      ) {
        return null;
      }
      const startMarkerWidth = 1;
      const endMarkerWidth = 8;
      const startX = fromTask.x;
      const startY = fromTask.y + fromTask.height / 2;
      const stopX = toTask.x;
      const stopY = toTask.y + toTask.height / 2;
      const offset = this.getOffset();
      const roundness = 4;
      let distanceX = stopX - startX;
      let directionX = 1;
      if(distanceX <= 0){
        directionX = -1;
      }
      let distanceY = stopY - startY;
      let directionY = 1;
      if (distanceY < 0) {
        directionY = -1;
      }
      let points = `
            M ${startX - startMarkerWidth} ${startY}
      `;
      if(distanceX == 0){
        points += `L ${startX - offset} ${startY}`;
        let _turn = this.turnVert(startX - offset, startY, roundness, -1, directionY);
        points += `${_turn.cmd}
        L ${_turn.x}, ${stopY - roundness*directionY}
        `;
        _turn = this.turnHor(_turn.x, stopY - roundness*directionY, roundness, 1, directionY);
        points += `${_turn.cmd}
        L ${stopX - endMarkerWidth}, ${stopY}
        `;

      }else if(directionX > 0){
        points +=`L ${startX - offset},${startY} `;
        let _turn = this.turnVert(startX - offset, startY, roundness, -1, directionY);
        points += `${_turn.cmd}
            L ${_turn.x},${_turn.y + (distanceY*directionY)/2 * directionY - roundness * directionY}
        `;

        _turn = this.turnHor(_turn.x, _turn.y + (distanceY*directionY)/2 * directionY - roundness * directionY, roundness, directionX, directionY);
        points += `${_turn.cmd}
        ${this.getHLineCommand(_turn.x, _turn.y, stopX - endMarkerWidth, stopY, roundness)}
        `;
      }else{
        points += this.getHLineCommand(startX, startY, stopX - offset, stopY - distanceY/2 - roundness*directionY, roundness);
        let _turn = this.turnVert(stopX - offset, stopY - distanceY/2 - roundness*directionY, roundness, -1, directionY);
        points += `${_turn.cmd}
            L ${_turn.x},${stopY - roundness * directionY}
        `;

        _turn = this.turnHor(_turn.x, stopY - roundness * directionY, roundness, 1, directionY);
        points += `${_turn.cmd}
        L ${stopX - endMarkerWidth},${stopY}
        `;
      }

      return points;
    },
    /**
     * Get path points
     *
     * @param {any} fromTaskId
     * @param {any} toTaskId
     * @returns {string}
     */
    getPointsFinish2Finish(fromTaskId, toTaskId) {
      const fromTask = this.root.getTask(fromTaskId);
      const toTask = this.root.getTask(toTaskId);
      if (
        fromTask === null ||
        toTask === null ||
        !this.root.isTaskVisible(toTask) ||
        !this.root.isTaskVisible(fromTask)
      ) {
        return null;
      }
      const startMarkerWidth = 1;
      const endMarkerWidth = 8;
      const startX = fromTask.x + fromTask.width;
      const startY = fromTask.y + fromTask.height / 2;
      const stopX = toTask.x + toTask.width;
      const stopY = toTask.y + toTask.height / 2;
      const offset = this.getOffset();
      const roundness = 4;
      let distanceX = stopX - startX;
      let directionX = 1;
      if(distanceX - (offset + 2 * roundness) <= 0){
        directionX = -1;
      }
      let distanceY = stopY - startY;
      let directionY = 1;
      if (distanceY < 0) {
        directionY = -1;
      }
      let points = `
            M ${startX + startMarkerWidth} ${startY}
      `;
      if(distanceX == 0){
        points += `L ${startX + offset} ${startY}`;
        let _turn = this.turnVert(startX + offset, startY, roundness, 1, directionY);
        points += `${_turn.cmd}
        L ${_turn.x}, ${stopY - roundness*directionY}
        `;
        _turn = this.turnHor(_turn.x, stopY - roundness*directionY, roundness, -1, directionY);
        points += `${_turn.cmd}
        L ${stopX + endMarkerWidth}, ${stopY}
        `;
      } else if(directionX > 0){
        points += `${this.getHLineCommand(startX, startY, stopX + offset - roundness, stopY - distanceY/2 - roundness*directionY, roundness)}`;
        // points +=`L ${startX + offset},${startY} `;
        let _turn = this.turnVert(stopX + offset - roundness, stopY - distanceY/2 - roundness*directionY, roundness, directionX, directionY);
        points += `${_turn.cmd}
        L ${_turn.x}, ${stopY - roundness*directionY}
        `;
        _turn = this.turnHor(_turn.x, stopY - roundness*directionY, roundness, -1, directionY);
        points += `${_turn.cmd}
        L ${stopX + endMarkerWidth}, ${stopY}`
      }else{
        points += `L ${startX + offset - roundness} ${startY}`;
        let _turn = this.turnVert(startX + offset - roundness, startY, roundness, 1, directionY);
        points += `${_turn.cmd}
        L ${_turn.x}, ${stopY - distanceY/2 - roundness*directionY}
        `;
        _turn = this.turnHor(_turn.x, stopY - distanceY/2 - roundness*directionY, roundness, directionX, directionY);
        points += `${_turn.cmd}
        ${this.getHLineCommand(_turn.x, _turn.y, stopX + endMarkerWidth, stopY, roundness)}
        `;
      }

      return points;
    },
    /**
     * Get path points
     *
     * @param {any} fromTaskId
     * @param {any} toTaskId
     * @returns {string}
     */
    getPointsFinish2Start(fromTaskId, toTaskId) {
      const fromTask = this.root.getTask(fromTaskId);
      const toTask = this.root.getTask(toTaskId);
      if (
        fromTask === null ||
        toTask === null ||
        !this.root.isTaskVisible(toTask) ||
        !this.root.isTaskVisible(fromTask)
      ) {
        return null;
      }
      const startMarkerWidth = 1;
      const endMarkerWidth = 8;
      const startX = fromTask.x + fromTask.width;
      const startY = fromTask.y + fromTask.height / 2;
      const stopX = toTask.x;
      const stopY = toTask.y + toTask.height / 2;
      const distanceX = stopX - startX;
      let distanceY;
      let yMultiplier = 1;
      if (stopY >= startY) {
        distanceY = stopY - startY;
      } else {
        distanceY = startY - stopY;
        yMultiplier = -1;
      }
      const offset = this.getOffset();
      const roundness = 4;
      const isBefore = distanceX <= offset + roundness;
      let points = `M ${startX + startMarkerWidth} ${startY}
          L ${startX + offset},${startY} `;
      if (isBefore) {
        points += `Q ${startX + offset + roundness},${startY} ${startX + offset + roundness},${startY +
          roundness * yMultiplier}
            L ${startX + offset + roundness},${startY + (distanceY * yMultiplier) / 2 - roundness * yMultiplier}
            Q ${startX + offset + roundness},${startY + (distanceY * yMultiplier) / 2} ${startX + offset},${startY +
          (distanceY * yMultiplier) / 2}
            L ${startX - offset + distanceX},${startY + (distanceY * yMultiplier) / 2}
            Q ${startX - offset + distanceX - roundness},${startY + (distanceY * yMultiplier) / 2} ${startX -
          offset +
          distanceX -
          roundness},${startY + (distanceY * yMultiplier) / 2 + roundness * yMultiplier}
            L ${startX - offset + distanceX - roundness},${stopY - roundness * yMultiplier}
            Q ${startX - offset + distanceX - roundness},${stopY} ${startX - offset + distanceX},${stopY}
            L ${stopX - endMarkerWidth},${stopY}`;
      } else {
        points += `L ${startX + distanceX / 2 - roundness},${startY}
            Q ${startX + distanceX / 2},${startY} ${startX + distanceX / 2},${startY + roundness * yMultiplier}
            L ${startX + distanceX / 2},${stopY - roundness * yMultiplier}
            Q ${startX + distanceX / 2},${stopY} ${startX + distanceX / 2 + roundness},${stopY}
            L ${stopX - endMarkerWidth},${stopY}`;
      }
      return points;
    }
  },
  computed: {
    /**
     * Get tasks which are dependent on other tasks
     *
     * @returns {array}
     */
    dependencyTasks() {
      return this.tasks
        .filter(task => typeof task.dependentOn !== 'undefined')
        .map(task => {
          task.dependencyLines = task.dependentOn.map(dependencyCfg => {
            if(dependencyCfg && dependencyCfg != undefined){
              let id = dependencyCfg.id;
              let type = dependencyCfg.type;
              if(type == 'finish2start'){
                let lineColor = this.root.style['chart-row-bar-polygon'].stroke;
                // let lineColor = '#1a2c5b';
                if(this.root.getTask(id).style && this.root.getTask(id).style.base && this.root.getTask(id).style.base.fill){
                  lineColor = this.root.getTask(id).style.base.fill;
                }

                return { points: this.getPointsFinish2Start(id, task.id), lineColor:  lineColor};
              }
              if(type == 'start2start'){
                let lineColor = this.root.style['chart-row-bar-polygon'].stroke;
                // let lineColor = '#1a2c5b';
                if(this.root.getTask(id).style && this.root.getTask(id).style.base && this.root.getTask(id).style.base.fill){
                  lineColor = this.root.getTask(id).style.base.fill;
                }

                return { points: this.getPointsStart2Start(id, task.id), lineColor:  lineColor};
              }
              if(type == 'finish2finish'){
                let lineColor = this.root.style['chart-row-bar-polygon'].stroke;
                // let lineColor = '#1a2c5b';
                if(this.root.getTask(id).style && this.root.getTask(id).style.base && this.root.getTask(id).style.base.fill){
                  lineColor = this.root.getTask(id).style.base.fill;
                }

                return { points: this.getPointsFinish2Finish(id, task.id), lineColor:  lineColor};
              }
            }
            return {};
          });
          return task;
        })
        .filter(task => task.dependencyLines.points !== null);
    }
  }
};
</script>
