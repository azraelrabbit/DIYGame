<template>
  <div id="pictureEdit">
    <div class="pictureEditContainer" v-if="imgSrc.length != 0">
      <div class="canvasContainer" @mousedown="editCanvas($event)">
        <canvas
          class="pictureEditCanvas"
          v-bind:width="canvasDimension.width"
          v-bind:height="canvasDimension.height"
          v-bind:style="canvasDimension"
        ></canvas>
        <div
          class="cropFrame"
          v-if="mode=='crop'"
          @mousedown.stop="moveCropFrame($event)"
          v-bind:style="cropFrameDimension"
        ></div>
      </div>
      <div class="pictureEdit_toolKit">
        <div class="resize">
          <input type="range" min="0.1" max="3" step="0.05" v-model="scale" @input="resize($event)">
        </div>
        <div class="drag" @click="mode='drag'">drag</div>
        <div class="crop" @click="mode='crop'">crop</div>
        <div class="erase" @click="mode='erase'">erase</div>
        <div class="undo" @click="undo()">undo</div>
        <div class="reset" @click="reset()">reset</div>
        <div class="toolState">
          <div v-if="mode == 'crop'">
            <button @click="cutCanvas()">cut</button>
            <button @click="removeCropFrame()">remove</button>
          </div>
          <div v-else-if="mode == 'erase'">
            <span>small</span>
            <span>middle</span>
            <span>big</span>
          </div>
        </div>
      </div>
    </div>
    <br>
    <button @click="addPicture()" v-if="imgSrc.length == 0">添加图片</button>
  </div>
</template>

<script>
import { isTSExpressionWithTypeArguments } from "@babel/types";
import { posStringToNumber } from "../../../lib/posStringToNumber";
import { transform } from "@babel/core";

export default {
  name: "pictureEdit",
  data: function() {
    return {
      imgSrc: "",
      historyData: [],
      canvasDimension: {
        width: 700,
        height: 400,
        top: 0,
        left: 0,
        transform: "scale(1)"
      },
      scale: 1,
      ratio: 1,
      cropFrameDimension: {
        width: 0,
        height: 0,
        top: 0,
        left: 0
      },
      mode: "drag"
    };
  },
  computed: {
    context: function() {
      return document.querySelector(".pictureEditCanvas").getContext("2d");
    },
    canvasContainer: function() {
      return this.imgSrc.length != 0
        ? document.getElementsByClassName("canvasContainer")[0]
        : null;
    },
    canvasDOM: function() {
      return this.imgSrc.length != 0
        ? document.getElementsByClassName("pictureEditCanvas")[0]
        : null;
    },
  },
  methods: {
    //放大缩小
    resize(event) {
      this.canvasDimension.transform = `scale(${event.target.value})`;
    },
    //重置
    reset() {
      this.context.clearRect(
        0,
        0,
        this.canvasDimension.width,
        this.canvasDimension.height
      );
      this.initCanvas();
    },
    //撤回
    undo() {
      if (this.historyData.length < 2) return;
      this.historyData.shift();
      let { imageData } = this.historyData[0];
      this.drawCanvas(imageData);
    },
    //处理canvas
    editCanvas(event) {
      switch (this.mode) {
        case "drag":
          return this.moveCanvas(event);
          break;
        case "crop":
          return this.drawCropFrame(event);
          break;
        case "erase":
          break;
      }
    },
    //移除cropFrame
    removeCropFrame() {
      this.cropFrameDimension.width = 0;
      this.cropFrameDimension.height = 0;
      this.mode = "drag";
    },
    //剪切canvas
    cutCanvas() {
      let { width, height } = posStringToNumber(this.cropFrameDimension);
      let { widthDiff, heightDiff } = this.cropFrameCanvasPosDiff();
      let x = widthDiff / this.scale;
      let y = heightDiff / this.scale;
      let newImageData = this.context.getImageData(
        x,
        y,
        width / this.scale,
        height / this.scale
      );
      this.drawCanvas(newImageData, x, y);
      this.historyUpdate();
      this.removeCropFrame();
    },
    //移动cropFrame
    moveCropFrame(event) {
      let self = this;
      let { xDiff, yDiff } = this.mouseCropFramePosDiff(event);
      self.canvasContainer.addEventListener("mousemove", startMove);
      function startMove(event) {
        if (event.buttons == 0) {
          self.canvasContainer.removeEventListener("mousemove", startMove);
        }
        self.cropFrameDimension.top =
          event.clientY - self.canvasContainerRectPos().top - yDiff + "px";
        self.cropFrameDimension.left =
          event.clientX - self.canvasContainerRectPos().left - xDiff + "px";
      }
    },
    //画cropFrame
    drawCropFrame(event) {
      let self = this;
      let originalMouseX = event.clientX;
      let originalMouseY = event.clientY;
      this.canvasContainer.addEventListener("mousemove", startDraw);
      let originalX = originalMouseX - this.canvasContainerRectPos().left;
      let originalY = originalMouseY - this.canvasContainerRectPos().top;

      function startDraw(event) {
        if (event.buttons == 0) {
          self.canvasContainer.removeEventListener("mousemove", startDraw);
        }
        let xDiff = event.clientX - originalMouseX;
        let yDiff = event.clientY - originalMouseY;
        self.cropFrameDimension.width = Math.abs(xDiff) + "px";
        self.cropFrameDimension.height = Math.abs(yDiff) + "px";
        self.cropFrameDimension.top =
          yDiff > 0 ? originalY + "px" : originalY + yDiff + "px";
        self.cropFrameDimension.left =
          xDiff > 0 ? originalX + "px" : originalX + xDiff + "px";
      }
    },
    //拖动canvas
    moveCanvas(event) {
      let self = this;
      let { widthDiff, heightDiff } = this.mouseImgPosDiff(event);
      self.canvasContainer.addEventListener("mousemove", startMove);
      function startMove(event) {
        if (event.buttons == 0) {
          self.canvasContainer.removeEventListener("mousemove", startMove);
        }
        self.canvasDimension.top =
          event.clientY -
          self.canvasContainerRectPos().top -
          heightDiff +
          self.scaleDiff().yCompensate +
          "px";
        self.canvasDimension.left =
          event.clientX -
          self.canvasContainerRectPos().left -
          widthDiff +
          self.scaleDiff().xCompensate +
          "px";
      }
    },
    //获取位置信息
    canvasContainerRectPos: function() {
      return this.canvasContainer.getBoundingClientRect();
    },
    canvasRectPos: function() {
      return this.canvasDOM.getBoundingClientRect();
    },
    mouseImgPosDiff: function(originalEvent) {
      return {
        widthDiff: originalEvent.clientX - this.canvasRectPos().left,
        heightDiff: originalEvent.clientY - this.canvasRectPos().top
      };
    },
    cropFrameCanvasPosDiff: function() {
      let cropFramePos = document
        .getElementsByClassName("cropFrame")[0]
        .getBoundingClientRect();
      return {
        widthDiff: cropFramePos.left - this.canvasRectPos().left,
        heightDiff: cropFramePos.top - this.canvasRectPos().top
      };
    },
    mouseCropFramePosDiff: function(originalEvent) {
      let cropFramePos = document
        .getElementsByClassName("cropFrame")[0]
        .getBoundingClientRect();
      return {
        xDiff: originalEvent.clientX - cropFramePos.left,
        yDiff: originalEvent.clientY - cropFramePos.top
      };
    },
    scaleDiff: function() {
      //scale 改变后，top，left数据没有发生变化，但实际上是变化了的，需要“补偿“变化的部分
      let { top, left } = posStringToNumber(this.canvasDimension);
      return {
        xCompensate:
          left -
          (this.canvasRectPos().left - this.canvasContainerRectPos().left),
        yCompensate:
          top - (this.canvasRectPos().top - this.canvasContainerRectPos().top)
      };
    },
    //首次加载图片
    initCanvas() {
      if (this.imgSrc.length == 0) return;
      //首次加载
      let img = new Image();
      img.src = this.imgSrc;
      img.onload = () => {
        let imgWidth = img.width;
        let imgHeight = img.height;
        let { width, height } = this.canvasDimension;
        let ratio = (this.ratio =
          width / imgWidth < height / imgHeight
            ? width / imgWidth
            : height / imgHeight);

        //保证整个图片能囊括在canva里面，并自动填满canvas。
        this.context.drawImage(img, 0, 0, imgWidth * ratio, imgHeight * ratio);
        this.historyUpdate(0, 0);
      };
    },
    //根据数据来画画
    drawCanvas(imgData, x = 0, y = 0) {
      this.context.clearRect(
        0,
        0,
        this.canvasDimension.width,
        this.canvasDimension.height
      );
      this.context.putImageData(imgData, x, y);
    },
    //把canvas数据放到history中
    historyUpdate() {
      //获得全canvas数据
      //但指明要样式的未知
      let { width, height } = this.canvasDimension;
      let newImageData = this.context.getImageData(0, 0, width, height);
      this.historyData.unshift({
        imageData: newImageData
      });
      if (this.historyData.length > 10) {
        this.historyData.pop();
      }
    },
    //添加图片
    addPicture() {
      let input = document.createElement("input");
      input.setAttribute("type", "file");
      input.setAttribute("accept", "image/*");
      input.addEventListener("change", () => {
        if (input.files[0] == null) {
          console.log("failed to load img");
        } else {
          let reader = new FileReader();
          reader.addEventListener("load", () => {
            this.imgSrc = reader.result;
            this.initCanvas();
          });
          reader.readAsDataURL(input.files[0]);
        }
      });
      input.click();
      input.remove();
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
$toolKit-list: resize drag crop erase undo reset toolState;

%hover-effect {
  cursor: pointer;
  opacity: 0.8;
}
%move-effect {
  cursor: grab;
}

.canvasContainer {
  width: 100%;
  height: 400px;
  border: 0.5px solid lightblue;
  position: relative;
  overflow: scroll;

  canvas {
    position: absolute;
  }

  .cropFrame {
    position: absolute;
    border: 2px black dashed;

    &:hover {
      @extend %move-effect;
      border-color: red;
    }
  }
}
.pictureEdit_toolKit {
  div {
    display: inline-block;
    border: 1px solid lightblue;
    padding: 0 3px;
    border-top: none;
  }
  @each $className in $toolKit-list {
    @if $className != resize and $className != toolState {
      .#{$className}:hover {
        @extend %hover-effect;
      }
    }
  }
  .toolState span {
    border: 1px solid grey;
    margin: 2px;
    padding: 0 2px;
  }
  .toolState span {
    &:hover {
      @extend %hover-effect;
    }
  }
}
</style>