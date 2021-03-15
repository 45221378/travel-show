<template>
  <div class="">
    <div>
      <img title="查看批注图片" crossOrigin="anonymous" :src="imgSrc + '?time=' + new Date().valueOf()" @click="showImgTrue">
      <!-- + '?time=' + new Date().valueOf() -->
      <img title="查看批注图片" crossOrigin="anonymous" :src="simageURL+'?time=' + new Date().valueOf()" style="display:none">
    </div>
    <div class="viewerAndAnnotate" v-if="showImg">
      <div class="viewerWrap" ref="correctImgWrap">
        <div @click="showImgFalse" role="button" class="viewer-button viewer-close" data-viewer-action="mix"></div>
        <div ref="imgElement" class="imgAndAnnotateWrap" :style="'left:'+ WrapStyle.imgLeft+'px; top:'+ WrapStyle.imgTop+'px; width:' +WrapStyle.imgWidth +'px; height:'+WrapStyle.imgHeight+'px'" @mousewheel="rollImg">
          <img ref="imgHolder" @mousedown="dragImgMove" @mouseup="removeDrag" crossOrigin="anonymous" :title="showPercent" width="100%" height="100%" :style="imgStyle" class="showIMG" :src="simageURL+'?time=' + new Date().valueOf()">
          <div ref="canvasWrap" v-show="editFlag" class="annotateWrap">
            <canvas ref="cNode" id="cNode" />
          </div>
        </div>
        <textarea v-if="showTextArea" v-model="text" @blur="whriteTextToCanvas" :style="'left:'+textLeft1+'px; top:'+textTop1+'px; font-size:'+textFontSize+'px; color:'+textColor+';font-family:'+textFontFamily" class="textarea" rows="3" ref="textInputEl"></textarea>
      </div>

      <div class="handle-footer">
        <!-- 操作图片的工具栏 -->
        <div class="handle-toolbar">
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
              <el-color-picker size="mini" v-model="textColor" :predefine="predefineColors" @change="resetColor">
              </el-color-picker>
            </div>
            <li title="标记" :class="drawType=='pen'?'iconfont icon-bi checkLIne':'iconfont icon-bi'" @click="switchAnnotate('pen')"></li>
            <li title="框选" :class="drawType=='rectangle'?'iconfont icon-zhengfangxing checkLIne':'iconfont icon-zhengfangxing'" @click="switchAnnotate('rectangle')"></li>
            <li title="字体" :class="drawType=='fontsize'?'iconfont icon-t checkLIne':'iconfont icon-t'" @click="switchAnnotate('fontsize')">
              <div class="selectfont" @change="getfontfamily" v-show="drawType=='fontsize'">
                <select>
                  <option v-for="(item,index) in optionfont" :key="index" :value="item.value">{{item.name}}</option>
                </select>
              </div>
              <div class="selectsize" @change="getfontsize" v-show="drawType=='fontsize'">
                <select>
                  <option v-for="(item,index) in optionfontsize" :key="index" :value="item.value">{{item.name}}</option>
                </select>
              </div>
            </li>
            <li title="撤销" :class="drawType=='resetlast'?'iconfont icon-chexiao checkLIne':'iconfont icon-chexiao'" @click="switchAnnotate('resetlast')"></li>
            <li title="清空" :class="drawType=='clear'?'iconfont icon-clear checkLIne':'iconfont icon-clear'" @click="switchAnnotate('clear')"></li>

            <!-- <li title="橡皮擦" :class="drawType==3?'iconfont icon-rubber checkLIne':'iconfont icon-rubber'" @click="switchAnnotate(3)"></li> -->
            <!-- <li title="删除" :class="drawType=='delete'?'iconfont icon-trash checkLIne':'iconfont icon-trash'" @click="switchAnnotate('delete')"></li> -->
            <!-- <li title="保存" class="iconfont icon-baocun1" @click="sendCanvas"></li> -->
            <!-- <div :class="imgbtndisable?'img-btn img-btndisable':'img-btn'" @click="sendCanvas">
              <p>保存批注</p>
            </div> -->
            <el-popover placement="top" width="160" v-model="visible">
              <p>若保存成功，将无法撤销，请确认是否保存批注？</p>
              <div style="text-align: right; margin: 0">
                <el-button size="mini" type="text" @click="visible = false">取消</el-button>
                <el-button type="primary" size="mini" @click="sendCanvas">确定</el-button>
              </div>
              <el-button class="savebtn" slot="reference">保存批注</el-button>
            </el-popover>
          </ul>
        </div>
        <p class="tips">鼠标滚动可放大缩小图片，按住空格可拖动图片</p>
      </div>
    </div>

  </div>
</template>

<script>
import { fabric } from "fabric";
import filter from "lodash/filter";
export default {
  name: "app-handleImg",
  props: [
    "imgSrc",
    "simageURL",
    "questionData",
    "questionNum",
    "sectionId",
    "scanDataId",
  ],
  data() {
    return {
      isCanvasRendered: false,
      showImg: false,
      scaleNum: 1,
      rotateNum: 0,
      editFlag: false,
      imgElement: null,
      canvas: null, //画板全局属性
      textColor: "#ff0000",
      textFontSize: 14,
      textFontFamily: "Microsoft Yahei",
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
      imgbtndisable: false, //必须有笔记了才能进行保存
      textbox: null,
      visible: false,
      // imgSrc: require('../../assets/images/electron/book-chemistry-img.png')
      // 添加字体所需参数
      inputArea: {}, //鼠标进入画布的时候，textarea框的位置
      textLeft1: undefined, //鼠标失去焦点的时候，文字渲染画布的位置
      textTop1: undefined,
      textLeft: undefined, //鼠标失去焦点的时候，文字渲染画布的位置
      textTop: undefined,
      showTextArea: false,
      text: "",
      optionfont: [
        { name: "微软雅黑", value: "Microsoft YaHei" },
        { name: "宋体", value: "SimHei" },
        { name: "黑体", value: "SimSun" },
        { name: "楷体", value: "KaiTi" },
      ],
      optionfontsize: [
        { name: "14px", value: 14 },
        { name: "16px", value: 16 },
        { name: "20px", value: 20 },
        { name: "24px", value: 24 },
        { name: "28px", value: 28 },
      ],
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
      if (val) {
        this.canvas.freeDrawingBrush.color = val;
      } else {
        this.canvas.freeDrawingBrush.color = "#ff0000";
      }
    },
    brushWidth(val) {
      console.log(val);
      if (val) {
        this.canvas.freeDrawingBrush.width = val;
      }
    },
    showImg(val) {
      if (!val) {
        window.removeEventListener("keydown", this.setDragEvent);
        window.removeEventListener("keyup", this.removeDragEvent);
        window.removeEventListener("resize", this.onResize, true);
      } else {
        window.addEventListener("keydown", this.setDragEvent);
        window.addEventListener("keyup", this.removeDragEvent);
        window.addEventListener("resize", this.onResize, true);
      }
    },
    // 监听是否出现笔迹，是否可以保存
  },
  methods: {
    resetColor() {
      if (this.textColor == null) {
        this.textColor = "#ff0000";
      }
    },
    // 如果弹框打开，则需要给body加上overflow：hidden的属性，否则背景还是可以滚动的
    showImgTrue() {
      this.showImg = true;
      document.querySelector("body").setAttribute("style", "overflow:hidden;");
      //计算图片属性
      setTimeout(() => {
        this.showImg && this.initCanvas();
        this.canvas && this.canvas.clear();
        this.pressImg();
        this.editImg();
      }, 50);
    },
    editImg() {
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
    showImgFalse(e) {
      if (e.currentTarget === e.target) {
        this.visible = false;
        this.showImg = false;
        this.editFlag = false;
        this.drawType = "pen";
        this.showPercent = 100;
        //清空字体记录
        this.textFontSize = 14;
        (this.textFontFamily = "Microsoft Yahei"),
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
    // 图片拖动方法
    dragImgMove(e) {
      e.preventDefault();
      e.stopPropagation();
      this.startPoint = {
        x: e.clientX,
        y: e.clientY,
      };
      let parentBox = e.target.offsetParent;
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
        //空格会与写字的空格冲突
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
      this.pressImg();
    },
    // 压缩图片
    pressImg() {
      const { naturalHeight, naturalWidth, src } = this.$refs.imgHolder;
      const imgType = this.simageURL.split(".").pop();
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
        this.imgOptions = {
          imgWidth: width,
          imgHeight: height,
          src: pressdSrc,
          showPercent: this.showPercent,
        }
        this.resetImg(this.imgOptions);
      };
      // console.log(src)
      img.src =
        src.indexOf("http") === 0 && src.indexOf("?time=") === -1
          ? `${src}?time=${new Date().valueOf() - 100000}`
          : src;
      img.crossOrigin = "anonymous";
    },
    // 计算图片属性
    resetImg(imgOptions) {
      const {
        imgHeight: nowHeight,
        imgWidth: nowWidth,
        showPercent: nowPercent,
        src,
        naturalHeight,
        naturalWidth,
      } = imgOptions;

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
    },
    // 缩放图片
    imgScale(e) {
      const naturalHeight = this.imgOptions.imgHeight;
      let { imgWidth, imgHeight } = this.WrapStyle;
      let { showPercent } = this.imgOptions;
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
    // 初始化canvas
    initCanvas() {
      //获取图片的宽高，并且赋值给canvas
      this.imgElement = this.$refs.imgHolder;
      const { naturalHeight, naturalWidth } = this.$refs.imgHolder;
      const { width, height } = this.resetImgWidthAndHeight(
        naturalWidth,
        naturalHeight
      );
      this.isCanvasRendered = true;
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
        // console.log(options);
        if (!options.pointer.x) {
          var xy = this.transformMouse(options.e.offsetX, options.e.offsetY);
        } else {
          var xy = this.transformMouse(options.pointer.x, options.pointer.y);
        }
        this.mouseFrom.x = xy.x;
        this.mouseFrom.y = xy.y;
        this.doDrawing = true;
        console.log(this.drawType);
        if (this.drawType == "fontsize") {
          this.showTextArea = true;
          setTimeout(() => {
            this.textLeft1 = options.e.clientX;
            this.textTop1 = options.e.clientY;
            this.showTextArea && this.$refs.textInputEl.focus();
            this.showTextArea &&
              (this.textleft = JSON.parse(JSON.stringify(this.mouseFrom.x)));
            this.showTextArea &&
              (this.textTop = JSON.parse(JSON.stringify(this.mouseFrom.y)));
          }, 50);
        }
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
        // if (this.drawType != "fontsize") {
        // }
        this.drawing();
      });
      canvas.on("selection:created", function (e) {
        //单选删除
        canvas.remove(e.target);
      });
      this.canvas = canvas;
    },
    // 切换批注功能
    switchAnnotate(type) {
      this.canvas.isDrawingMode = false;
      this.drawType = type;
      this.showTextArea = false;

      if (type == "pen") {
        this.canvas.isDrawingMode = true;
      } else if (type == "rectangle") {
        this.canvas.defaultCursor = "crosshair";
        this.canvas.skipTargetFind = true; //画板元素不能被选中
        this.canvas.selection = false; //画板不显示选中
      } else if (type == "fontsize") {
        this.canvas.defaultCursor = "text";
        this.canvas.skipTargetFind = true; //画板元素不能被选中
        this.canvas.selection = false; //画板不显示选中
        // this.showTextArea = true;
      } else if (type == "clear") {
        this.imgToCanvas();
      } else if (type == "delete") {
        this.canvas.skipTargetFind = false; //画板元素不能被选中
        this.canvas.selection = true; //画板不显示选中
        this.canvas.selectable = true;
      } else if (type == "resetlast") {
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
      // console.log(this.drawingObject);
      // if (this.drawingObject) {
      //   this.canvas.remove(this.drawingObject);
      // }
      var canvasObject = null;
      switch (this.drawType) {
        case "pen": //随意画
          var left = this.mouseFrom.x,
            top = this.mouseFrom.y;
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
            left: left,
            top: top,
            stroke: this.textColor,
            strokeWidth: this.brushWidth,
            fill: "rgba(255, 255, 255, 0)",
          });
          break;
        case "fontsize":
          if (this.text != "") {
            canvasObject = new fabric.Text(this.text, {
              left: this.textleft,
              top: this.textTop,
            });
            canvasObject.height = this.textFontSize;
            canvasObject.padding = 0;
            canvasObject.fontSize = this.textFontSize;
            canvasObject.fontFamily = this.textFontFamily;
            canvasObject.fill = this.textColor;
            canvasObject.selectable = true;
            canvasObject.evented = true;
            canvasObject.absolutePositioned = true;
            this.text = "";
            this.textLeft = undefined;
            this.textTop = undefined;
          }
          break;
        case "delete": //删除功能
          break;
        default:
          break;
      }
      // console.log(canvasObject);
      if (canvasObject) {
        canvasObject.index = this.getCanvasObjectIndex();
        this.canvas.add(canvasObject);
        this.drawingObject = canvasObject;
      }
    },
    //失去焦点的时候，把文字写入canvas
    whriteTextToCanvas(e) {
      this.showTextArea = false;
      if (this.text == "") {
        this.textLeft = undefined;
        this.textTop = undefined;
      } else {
        this.drawing();
      }
    },
    getCanvasObjectIndex() {
      return this.canvasObjectIndex++;
    },
    // 删除某一次操作
    onDetlteInsert(e) {
      this.canvas.remove(e.target);
      this.canvas.renderAll();
    },
    // 撤销上一次操作,图片导入了canvsa,不能撤销图片,所以要length - 1
    undo() {
      const objects = this.canvas.getObjects();
      if (objects && objects.length > 1) {
        this.canvas.remove(objects[objects.length - 1]);
        this.canvas.renderAll();
      }
    },
    // 发送数据与后端进行交互
    sendCanvas() {
      // 第一个是img  从第二个开始的path 就是轨迹
      // console.log(this.canvasObjectIndex);

      // console.log(this.canvas._objects.length);
      // console.log(this.canvas.getObjects().length);
      if (this.canvas && this.canvas._objects.length > 1) {
        const dataUrl = this.canvas.toDataURL({
          format: "image/png",
          quality: 1,
        });
        // console.log(dataUrl);
        //   // 这一段是多余的代码，看看后期是否可以删掉
        let mark_data = [];
        this.questionData.map((item) => {
          if (item.question_num === this.questionNum) {
            item.subquestionInfo.map((it) => {
              mark_data.push({
                q_number: it.index,
                mark_type: it.analyzeType,
                q_score: it.score,
                s_answer: it.analyzeResult,
                q_answer: it.answer,
              });
            });
          }
        });
        //传base64给后端进行交互
        this.$post("/teacher/homework/mark/line", {
          data: {
            image_url: this.simageURL,
            mark_data: JSON.stringify(mark_data),
            tag_type: 1, //告知后端对图片的修改是写答案0，还是改批注1
            image_base64: dataUrl,
            scan_data_id: this.scanDataId,
            question_num: this.questionNum,
          },
          ignoreStatusCheck: true,
          loadingEl: this.$refs.correctImgWrap,
        }).then((data) => {
          // console.log(data);
          if (data.status_code == 10200) {
            this.$emit("changeimgSrc", {
              newimg: data.tmp_img_url,
              questionNum: this.questionNum,
              imageURL: this.imgSrc,
              tag_type: 1,
            });
            this.visible = false;
            this.showImg = false;
            this.editFlag = false;
            this.drawType = "pen";
            this.showPercent = 100;
            //清空字体记录

            this.textFontSize = 14;
            (this.textFontFamily = "Microsoft Yahei"),
              document.querySelector("body").removeAttribute("style");
          } else {
            this.$message.error(data.error);
          }
        });
      } else {
        this.visible = false;
      }
    },
    getfontfamily(e) {
      this.textFontFamily = e.target.value;
      this.$forceUpdate();
    },
    getfontsize(e) {
      this.textFontSize = e.target.value;
      this.$forceUpdate();
    },

    // 将base64转成blob对象方法
    transformBase64ToFile(dataUrl) {
      const arr = dataUrl.split(",");
      const bstr = atob(arr[1]);
      let n = bstr.length;
      const u8arr = new Uint8Array(n);
      while (n--) {
        u8arr[n] = bstr.charCodeAt(n);
      }
      const appendix = filter(
        this.props.appendix,
        (a) => a.imgIndex === this.state.currentIndex + 1
      )[0];
      const blob = new Blob([u8arr], {
        type: appendix.type,
      });
      const file = new File([blob], appendix.name, {
        type: appendix.type,
      });
      return {
        file,
        appendix,
      };
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
.viewerAndAnnotate {
  top: 0;
  left: 0;
  overflow: hidden;
  position: fixed;
  right: 0;
  bottom: 0;
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
          width: 100%;
          height: 100%;
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
    position: relative;
  }
  .img-btndisable {
    background: rgba($color: #cd5f62, $alpha: 0.7) !important;
    cursor: default !important;
  }
  .changebtn {
    height: 25px;
    line-height: 1px;
    padding: 0px 5px;
    margin: 0 0 0 10px;
    font-size: 12px;
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
    position: relative;
  }
  .iconfont .selectfont {
    position: absolute;
    top: -26px;
    left: -34px;
  }
  .iconfont .selectsize {
    position: absolute;
    top: -26px;
    left: 40px;
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
  .textarea {
    // border: none;
    border: 1px solid #000;
    background-color: rgba($color: #000000, $alpha: 0);
    outline: none;
    position: absolute;
    z-index: 99;
  }
  // 底部工具栏样式
  .handle-footer {
    bottom: 0;
    left: 0;
    // overflow: hidden;
    position: absolute;
    right: 0;
    text-align: center;
    .handle-toolbar {
      & > ul {
        display: inline-block;
        margin: 0 auto 5px;
        // overflow: hidden;
        padding: 3px 0;
        & > li {
          background-color: rgba(0, 0, 0, 0.5);
          border-radius: 50%;
          cursor: pointer;
          float: left;
          height: 24px;
          // overflow: hidden;
          transition: background-color 0.15s;
          width: 24px;
          &:hover {
            background-color: rgba(0, 0, 0, 0.8);
          }
          &::before {
            margin: 2px;
          }
          & + li {
            margin-left: 1px;
          }
        }
        & > .viewer-small {
          height: 18px;
          margin-bottom: 3px;
          margin-top: 3px;
          width: 18px;
          &::before {
            margin: -1px;
          }
        }
        & > .viewer-large {
          height: 30px;
          margin-bottom: -3px;
          margin-top: -3px;
          width: 30px;
          &::before {
            margin: 5px;
          }
        }
      }
    }
  }
  .savebtn {
    padding: 4px 10px;
    color: #fff;
    background-color: #cd5f62;
    border-color: #cd5f62;
    border-radius: 14px;
    margin-left: 10px;
    &:hover {
      background: #d77f81;
      border-color: #d77f81;
      color: #fff;
    }
  }
}
</style>

