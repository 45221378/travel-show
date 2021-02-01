<template>
  <div class="">
    <div>
      <img title="查看批注图片" crossOrigin="anonymous" :src="imgSrc" @click="showImgTrue">
    </div>
    <div class="hint-img" v-if="showImg">
      <div class="viewerWrap">
        <div @click="showImgFalse" role="button" class="viewer-button viewer-close" data-viewer-action="mix"></div>
        <div ref="imgElement" class="imgAndAnnotateWrap" :style="'left:'+ WrapStyle.imgLeft+'px; top:'+ WrapStyle.imgTop+'px; width:' +WrapStyle.imgWidth +'px; height:'+WrapStyle.imgHeight+'px'" @mousewheel="rollImg">
          <img @mousedown="dragImgMove" @mouseup="removeDrag" ref="imgHolder" crossOrigin="anonymous" :title="showPercent" width="100%" height="100%" :style="imgStyle" class="showIMG" :src="imgSrc">
          <div ref="canvasWrap" v-show="editFlag" class="annotateWrap">
            <canvas ref="cNode" id="cNode" />
          </div>
        </div>
      </div>

      <div class="viewer-footer">
        <!-- 操作图片的工具栏 -->
        <div class="viewer-toolbar" v-if="editFlag">
          <ul class="clearfix">
            <!-- 挂载的调节大小和颜色的区域模块 -->
            <div class="mount">
              <span class="clickarea" @click="checkLIne(2)">
                <i :class="brushWidth==2?'small checkLIne':'small'"></i>
              </span>
              <span class="clickarea" @click="checkLIne(5)">
                <i :class="brushWidth==5?'middle checkLIne':'middle'"></i>
              </span>
              <span class="clickarea" @click="checkLIne(10)">
                <i :class="brushWidth==10?'big checkLIne':'big'"></i>
              </span>
              <el-color-picker size="mini" v-model="textColor" :predefine="predefineColors">
              </el-color-picker>
            </div>
            <li title="标记" :class="drawType=='pen'?'iconfont icon-bi checkLIne':'iconfont icon-bi'" @click="switchAnnotate('pen')"></li>
            <li title="框选" :class="drawType=='rectangle'?'iconfont icon-zhengfangxing checkLIne':'iconfont icon-zhengfangxing'" @click="switchAnnotate('rectangle')"></li>
            <!-- <li title="橡皮擦" :class="drawType==3?'iconfont icon-rubber checkLIne':'iconfont icon-rubber'" @click="switchAnnotate(3)"></li> -->
            <li title="删除" :class="drawType=='delete'?'iconfont icon-trash checkLIne':'iconfont icon-trash'" @click="switchAnnotate('delete')"></li>
            <li title="撤销" :class="drawType==2?'iconfont icon-chexiao checkLIne':'iconfont icon-chexiao'" @click="switchAnnotate(2)"></li>
            <!-- <li title="保存" class="iconfont icon-baocun1" @click="sendCanvas"></li> -->
            <p class="img-btn" @click="sendCanvas">
              保存批注
            </p>
          </ul>
        </div>
        <p class="tips">鼠标滚动可放大缩小图片，按住空格可拖动图片</p>
      </div>
    </div>
  </div>
</template>

<script>
import { fabric } from "fabric";
export default {
  name: "app-handleImg",
  props: ["imgSrc"],
  data() {
    return {
      showImg: false,
      scaleNum: 1,
      rotateNum: 0,
      editFlag: false,
      imgElement: null,
      canvas: null, //画板全局属性
      textColor: "#ff0000",
      textFontSize: 14,
      brushWidth: 2, //画笔的粗细
      //颜色拾取器的默认选项
      predefineColors: [
        "#ff0000",
        "#92d050",
        "#ffff00",
        "#00b0f0",
        "#333333",
        "#000000",
      ],
      naturalHeight: 0, //画布的高度
      naturalWidth: 0, //画布的宽度
      WrapStyle: {
        //图片的属性，根据图片本身的大小来自定义
        imgLeft: 0,
        imgTop: 0,
        imgWidth: 0,
        imgHeight: 0,
      },
      imgOptions: {},
      showPercent: 100,
      drawType: "pen",
      drawingObject: null, //表示当前的绘制对象
      moveCount: 1, //绘制移动计数器
      mouseFrom: {},
      mouseTo: {},
      canvasObjectIndex: 0,
      startPoint: {},
      hasAddMouseListener: false,
      // imgSrc: require('../../assets/images/electron/book-chemistry-img.png')
    };
  },
  computed: {
    imgStyle() {
      const obj = {
        transform: `scale(${this.scaleNum}) rotate(${this.rotateNum}deg)`,
      };
      return obj;
    },
  },
  watch: {
    //监听颜色和笔画粗细的情况
    textColor(val) {
      console.log(val);
      if (val) {
        this.canvas.freeDrawingBrush.color = val;
      }
    },
    brushWidth(val) {
      console.log(val);
      if (val) {
        this.canvas.freeDrawingBrush.width = val;
      }
    },
  }, 
  methods: {
    // 如果弹框打开，则需要给body加上overflow：hidden的属性，否则背景还是可以滚动的
    showImgTrue() {
      this.showImg = true;
      document.querySelector("body").setAttribute("style", "overflow:hidden;");
      //计算图片属性
      this.$nextTick(() => {
        this.showImg && this.initCanvas();
        this.canvas && this.canvas.clear();
        this.pressImg();
      });
    },
    showImgFalse(e) {
      if (e.currentTarget === e.target) {
        this.showImg = false;
        this.editFlag = false;
        document.querySelector("body").removeAttribute("style");
      }
    },
    rollImg(event) {
      event.preventDefault();
      if (event.wheelDelta) {
        //判断浏览器IE，谷歌滑轮事件
        if (event.wheelDelta > 0) {
          //当滑轮向上滚动时
          if (this.scaleNum >= 3) {
            return false;
          } else {
            this.imgScale(event);
          }
        }
        if (event.wheelDelta < 0) {
          //当滑轮向下滚动时
          if (this.scaleNum <= 0.2) {
            return false;
          } else {
            this.imgScale(event);
          }
        }
      } else if (event.detail) {
        //Firefox滑轮事件
        if (event.detail > 0) {
          //当滑轮向上滚动时
          if (this.scaleNum >= 5) {
            return false;
          } else {
            this.imgScale(event);
          }
        }
        if (event.detail < 0) {
          //当滑轮向下滚动时
          if (this.scaleNum <= 0.6) {
            return false;
          } else {
            this.imgScale(event);
          }
        }
      }
    },
    edit() {
      if (this.editFlag) {
        this.editFlag = false;
        //消除canvas
      } else {
        this.editFlag = true;
        //初始化canvas
        this.$nextTick(() => {
          this.imgToCanvas();
          // 动态修改canvas的大小后，必须重新给出canvas的位置。
          this.mathPrecent();
          if (this.canvas.freeDrawingBrush) {
            // console.log("设置画笔大小");
            this.canvas.freeDrawingBrush.color = this.textColor;
            this.canvas.freeDrawingBrush.width = this.brushWidth;
          }
        });
      }
    },
    // 根据img计算出的precent,得出canvas的情况
    mathPrecent() {
      if (this.$refs.canvasWrap) {
        let objectCanvas = this.$refs.canvasWrap.getElementsByTagName("canvas");
        // console.log(objectCanvas);
        Object.keys(objectCanvas).map((item) => {
          objectCanvas[item].style.transform = `scale(${
            (this.showPercent * 10) / 1000
          })`;
          objectCanvas[item].style.transformOrigin = "left top";
        });
      }
    },
    // 根据图片的大小，初始化的时候根据情况设置canvas的大小
    resetImgWidthAndHeight(naturalWidth, naturalHeight) {
      const ratio = naturalWidth / naturalHeight;
      let width = naturalWidth;
      let height = naturalHeight;
      if (naturalWidth > 4032 || naturalHeight > 4032) {
        if (naturalWidth > naturalHeight) {
          width = 4032;
          height = 4032 / ratio;
        } else {
          height = 4032;
          width = 4032 * ratio;
        }
      }
      return { width, height };
    },
    // 初始化canvas
    initCanvas() {
      //获取图片的宽高，并且赋值给canvas
      this.imgElement = this.$refs.imgHolder;
      const { naturalHeight, naturalWidth } = this.$refs.imgHolder;
      const { width, height } = this.resetImgWidthAndHeight(
        naturalWidth,
        naturalHeight
      );
      this.$refs.cNode.width = width;
      this.$refs.cNode.height = height;
      const canvas = new fabric.Canvas("cNode", {
        isDrawingMode: true,
        interactive: false,
      });
      // console.log(canvas.freeDrawingBrush);
      if (canvas.freeDrawingBrush) {
        canvas.freeDrawingBrush.color = this.textColor;
        canvas.freeDrawingBrush.width = this.brushWidth;
      }
      //监听 canvas 事件
      //绑定画板事件
      canvas.on("mouse:down", (options) => {
        if (!options.pointer.x) {
          var xy = this.transformMouse(options.e.offsetX, options.e.offsetY);
        } else {
          var xy = this.transformMouse(options.pointer.x, options.pointer.y);
        }
        this.mouseFrom.x = xy.x;
        this.mouseFrom.y = xy.y;
        this.doDrawing = true;
      });
      canvas.on("mouse:up", (options) => {
        if (!options.pointer.x) {
          var xy = this.transformMouse(options.e.offsetX, options.e.offsetY);
        } else {
          var xy = this.transformMouse(options.pointer.x, options.pointer.y);
        }
        this.mouseTo.x = xy.x;
        this.mouseTo.y = xy.y;
        this.drawingObject = null;
        this.moveCount = 1;
        this.doDrawing = false;
      });
      canvas.on("mouse:move", (options) => {
        if (this.moveCount % 2 && !this.doDrawing) {
          //减少绘制频率
          return;
        }
        this.moveCount++;
        if (!options.pointer.x) {
          var xy = this.transformMouse(options.e.offsetX, options.e.offsetY);
        } else {
          var xy = this.transformMouse(options.pointer.x, options.pointer.y);
        }
        this.mouseTo.x = xy.x;
        this.mouseTo.y = xy.y;
        this.drawing();
      });
      canvas.on("selection:created", function (e) {
        //单选删除
        canvas.remove(e.target);
      });
      this.canvas = canvas;
    },
    // 图片拖动方法
    dragImgMove(e) {
      e.preventDefault();
      e.stopPropagation();;
      this.startPoint = {
        x: e.clientX,
        y: e.clientY,
      };
      let parentBox = e.target.offsetParent
      document.onmousemove = (event) => {
        const { clientX: x, clientY: y } = event;
        this.WrapStyle.imgLeft += x - this.startPoint.x;
        this.WrapStyle.imgTop += y - this.startPoint.y;
        this.startPoint.x = x;
        this.startPoint.y = y;
        this.$forceUpdate();
      };
      parentBox.onmouseleave = () => {
        document.onmousemove = null;
      };
      document.onmouseup = () => {
        document.onmousemove = null;
      };
    },
    removeDrag() {
      document.onmouseup = () => {
        document.onmousemove = null;
      };
    },
    setDragEvent(event) {
      const e = event || window.event || arguments.callee.caller.arguments[0];
      if (!e) return;
      const { keyCode } = e;
      if (
        keyCode === 32 &&
        this.editFlag &&
        this.$refs.canvasWrap &&
        this.$refs.canvasWrap.style.zIndex !== -1
      ) {
        this.$refs.canvasWrap.style.zIndex = -1;
      }
    },
    removeDragEvent(event) {
      const e = event || window.event || arguments.callee.caller.arguments[0];
      if (!e) return;
      const { keyCode } = e;
      if (
        keyCode === 32 &&
        this.editFlag &&
        this.$refs.canvasWrap &&
        this.$refs.canvasWrap.style.zIndex !== 1
      ) {
        this.$refs.canvasWrap.style.zIndex = 1;
        // this.$refs.imgHolder.removeEventListener('mousemove',this.drawImage,false)
      }
    },
    // 坐标转换
    transformMouse(mouseX, mouseY) {
      return {
        x: mouseX / window.zoom,
        y: mouseY / window.zoom,
      };
    },
    // 监听屏幕大小变化，屏幕大小变化，图片的宽高随之改变
    onResize() {
      // this.resetImg();
    },
    // 压缩图片
    pressImg() {
      const { naturalHeight, naturalWidth, src } = this.$refs.imgHolder;
      const imgType = this.imgSrc.split(".").pop();
      const ratio = naturalWidth / naturalHeight;
      const img = new Image();
      let pressdSrc = "";
      let width = naturalWidth;
      let height = naturalHeight;

      img.onload = () => {
        const canvas = document.createElement("canvas");
        const ctx = canvas.getContext("2d");
        if (naturalWidth > 4032 || naturalHeight > 4032) {
          if (naturalWidth > naturalHeight) {
            width = 4032;
            height = 4032 / ratio;
          } else {
            height = 4032;
            width = 4032 * ratio;
          }
        }
        canvas.width = width;
        canvas.height = height;
        img.setAttribute("crossOrigin", "Anonymous");

        ctx.drawImage(img, 0, 0, width, height);

        pressdSrc = canvas.toDataURL(
          `image/${imgType === "jpg" ? "jpeg" : imgType}`
        );
        // 将图片导入到canvas以前，图片的宽高需要重新计算，按可视窗口的比例，等比缩放，以便能够完全适应屏幕，边角不会超出屏幕
        this.options = {
          imgWidth: width,
          imgHeight: height,
          src: pressdSrc,
          showPercent: 100,
        };
        this.resetImg(this.options);
      };
      img.src =
        src.indexOf("http") === 0 && src.indexOf("?time=") === -1
          ? `${src}?time=${new Date().valueOf() - 100000}`
          : src;
      img.crossOrigin = "anonymous";
    },
    // 计算图片属性
    resetImg(options) {
      const {
        imgHeight: nowHeight,
        imgWidth: nowWidth,
        showPercent: nowPercent,
        naturalHeight,
        naturalWidth,
      } = options;

      let imgWidth = nowWidth;
      let imgHeight = nowHeight;
      let showPercent = nowPercent || 100;
      const widthEqualHeight = nowWidth / nowHeight;
      const windowWidthEqualHeight = window.innerWidth / window.innerHeight;

      if (widthEqualHeight >= windowWidthEqualHeight) {
        if (
          nowWidth > window.innerWidth * 0.8 ||
          (naturalWidth > window.innerWidth * 0.8 &&
            nowWidth <= window.innerWidth * 0.8)
        ) {
          imgWidth = window.innerWidth * 0.8;
          imgHeight = imgWidth / widthEqualHeight;
          showPercent = (
            ((imgWidth / (naturalWidth || nowWidth)) * 1000) /
            10
          ).toFixed(5);
        }
      } else if (
        nowHeight > window.innerHeight * 0.8 ||
        (naturalHeight > window.innerHeight * 0.8 &&
          nowHeight <= window.innerHeight * 0.8)
      ) {
        imgHeight = window.innerHeight * 0.8;
        imgWidth = imgHeight * widthEqualHeight;
        showPercent = (
          ((imgHeight / (naturalHeight || nowHeight)) * 1000) /
          10
        ).toFixed(5);
      }
      const imgLeft = (window.innerWidth - imgWidth) / 2 || 0;
      const imgTop = (window.innerHeight - imgHeight) / 2 || 0;

      this.showPercent = showPercent;

      this.WrapStyle = {
        imgLeft: imgLeft,
        imgTop: imgTop,
        imgWidth: imgWidth,
        imgHeight: imgHeight,
      };
      this.edit();
    },
    // 缩放图片
    imgScale(e) {
      const naturalHeight = this.options.imgHeight;
      let { imgWidth, imgHeight } = this.WrapStyle;
      let { showPercent } = this.options;
      let isAnnotated = true;
      let rotateAngle = 0;
      if (this.isWheeling) {
        return;
      }
      this.isWheeling = true;
      setTimeout(() => {
        this.isWheeling = false;
      }, 50);

      let delta = 1;
      if (e.deltaY) {
        delta = e.deltaY > 0 ? -1 : 1;
      } else if (e.wheelDelta) {
        delta = e.wheelDelta / 120;
      } else if (e.detail) {
        delta = e.detail > 0 ? -1 : 1;
      }

      let ratio = Number(delta * 0.1);
      if (ratio < 0) {
        ratio = 1 / (1 - ratio);
      } else {
        ratio = 1 + ratio;
      }
      imgWidth *= ratio;
      imgHeight *= ratio;
      showPercent = ((imgHeight / naturalHeight) * 1000) / 10;
      let changedLeft = 0;
      let changedTop = 0;
      const offsetX = isAnnotated
        ? (e.offsetX / e.target.clientWidth) * e.target.offsetParent.clientWidth
        : e.offsetX;
      const offsetY = isAnnotated
        ? (e.offsetY / e.target.clientHeight) *
          e.target.offsetParent.clientHeight
        : e.offsetY;
      if (rotateAngle % 360 === 90 || rotateAngle % 360 === -270) {
        changedLeft = imgWidth / 2 + imgHeight / 2 - offsetY * ratio;
        changedTop = imgHeight / 2 - imgWidth / 2 + offsetX * ratio;
      } else if (rotateAngle % 360 === 180) {
        changedLeft = imgWidth - offsetX * ratio;
        changedTop = imgHeight - offsetY * ratio;
      } else if (rotateAngle % 360 === 270 || rotateAngle % 360 === -90) {
        changedLeft = imgWidth / 2 - imgHeight / 2 + offsetY * ratio;
        changedTop = imgHeight / 2 + imgWidth / 2 - offsetX * ratio;
      } else {
        changedLeft = offsetX * ratio;
        changedTop = offsetY * ratio;
      }
      this.WrapStyle.imgLeft = e.clientX - changedLeft;
      this.WrapStyle.imgTop = e.clientY - changedTop;
      this.WrapStyle.imgHeight = imgHeight;
      this.WrapStyle.imgWidth = imgWidth;
      this.showPercent = showPercent;
      this.mathPrecent();
      this.$forceUpdate();
    },
    // 将图片导入到canvas
    imgToCanvas() {
      if (this.canvas && this.imgElement) {
        const img = new fabric.Image(this.$refs.imgHolder);
        this.canvas.clear();
        this.canvas.add(img);
        this.canvas.renderAll();
      }
    },
    // 选择画笔粗细
    checkLIne(lineWidth) {
      this.brushWidth = lineWidth;
    },
    // 切换批注功能
    switchAnnotate(type) {
      this.canvas.isDrawingMode = false;
      this.drawType = type;
      if (type == "pen") {
        this.canvas.isDrawingMode = true;
      } else if (type == "rectangle") {
        this.canvas.defaultCursor = "crosshair";
        this.canvas.skipTargetFind = true; //画板元素不能被选中
        this.canvas.selection = false; //画板不显示选中
      } else if (type == "delete") {
        this.canvas.skipTargetFind = false; //画板元素不能被选中
        this.canvas.selection = true; //画板不显示选中
        this.canvas.selectable = true;
      } else if (type == 2) {
        this.undo();
        // } else if (type == 3) {
        //   //橡皮擦功能
        //   const eraserBrush = new EraserBrush(this.canvas);
        //   eraserBrush.width = 10;
        //   eraserBrush.color = "#fff";
        //   this.canvas.freeDrawingBrush = eraserBrush;
        //   this.canvas.isDrawingMode = true;
      } else {
        this.canvas.skipTargetFind = true; //画板元素不能被选中
        this.canvas.selection = false; //画板不显示选中
      }
    },
    drawing() {
      if (this.drawingObject) {
        this.canvas.remove(this.drawingObject);
      }
      var canvasObject = null;
      switch (this.drawType) {
        case "pen": //随意画
          this.canvas.isDrawingMode = true;
          break;
        case "rectangle": //画矩形
          var path =
            "M " +
            this.mouseFrom.x +
            " " +
            this.mouseFrom.y +
            " L " +
            this.mouseTo.x +
            " " +
            this.mouseFrom.y +
            " L " +
            this.mouseTo.x +
            " " +
            this.mouseTo.y +
            " L " +
            this.mouseFrom.x +
            " " +
            this.mouseTo.y +
            " L " +
            this.mouseFrom.x +
            " " +
            this.mouseFrom.y +
            " z";
          canvasObject = new fabric.Path(path, {
            left: this.mouseFrom.x,
            top: this.mouseFrom.y,
            stroke: this.textColor,
            strokeWidth: this.brushWidth,
            fill: "rgba(255, 255, 255, 0)",
          });
          break;
        case "delete": //删除功能
          break;
        default:
          break;
      }
      // console.log(canvasObject)
      if (canvasObject) {
        canvasObject.index = this.getCanvasObjectIndex();
        this.canvas.add(canvasObject);
        this.drawingObject = canvasObject;
      }
      // console.log(this.drawingObject)
    },
    getCanvasObjectIndex() {
      return this.canvasObjectIndex++;
    },
    // 删除某一次操作
    onDetlteInsert(e) {
      // console.log(e)
      this.canvas.remove(e.target);
      this.canvas.renderAll();
    },
    // 撤销上一次操作,图片导入了canvsa,不能撤销图片,所以要length - 1
    undo() {
      const objects = this.canvas.getObjects();
      if (objects && objects.length > 1) {
        this.canvas.remove(objects[objects.length - 1]);
      }
    },
    // 发送数据与后端进行交互
    sendCanvas() {
      // 第一个是img  从第二个开始的path 就是轨迹
      // console.log(this.canvas._objects)
    },
  },
  mounted() {
    window.zoom = window.zoom ? window.zoom : 1;
    window.addEventListener("keydown", this.setDragEvent);
    window.addEventListener("keyup", this.removeDragEvent);
  },
};
</script>
 

<style lang="scss">
.hint-img {
  bottom: 0;
  left: 0;
  overflow: hidden;
  position: fixed;
  right: 0;
  top: 0;
  z-index: 2001;
  background: rgba(0, 0, 0, 0.5);
  .viewerWrap {
    overflow: hidden;
    position: relative;
    width: 100%;
    height: 100%;
    .imgAndAnnotateWrap {
      position: absolute;
      transform-origin: center;
      transition: transform 0.3s;
      .annotateWrap {
        position: absolute;
        left: 0;
        top: 0;
        width: 100%;
        height: 100%;
        & > div {
          width: 100% !important;
          height: 100% !important;
        }
      }
    }
  }
  .showIMG {
    cursor: pointer;
    object-fit: contain;
    width: auto;
    user-select: none;
  }

  .img-btn {
    font-size: 14px;
    color: #fff;
    margin: 0 0 0 10px;
    float: left;
    background: #cd5f62;
    border-radius: 15px;
    padding: 0 20px;
    height: 24px;
    line-height: 24px;
    user-select: none;
  }
  // 操作工具栏样式
  .mount {
    display: flex;
    justify-content: center;
    align-items: center;
    float: left;
    height: 26px;
    margin: 0 4px 0 0;
    .clickarea {
      height: 24px;
      width: 24px;
      display: flex;
      justify-content: center;
      align-items: center;
      &:hover {
        i {
          background: #00b0f0;
        }
      }
      i {
        border-radius: 50%;
        background: #fff;
        float: left;
        margin: 0 4px;
      }
      .small {
        width: 3px;
        height: 3px;
      }
      .middle {
        width: 5px;
        height: 5px;
      }
      .big {
        width: 10px;
        height: 10px;
      }
    }
  }
  .checkLIne {
    background: #00b0f0 !important;
  }
  .iconfont {
    color: #fff;
    text-align: center;
    line-height: 24px;
    font-size: 14px;
  }
  .viewer-toolbar {
    ul {
      margin: 0 auto !important;
    }
  }
  .tips {
    color: #fff;
    line-height: 24px;
    font-size: 14px;
    margin: 0 0 5px 0;
  }
}
</style>

