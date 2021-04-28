# travel-show
#### vue 批改作业功能

###### 主要功能就是一张img为底图，在底图上面对其操作涂鸦，框选，撤销，以及保存功能。
ps: 橡皮擦功能比较复杂，而且目前的实现方案就是用白色的覆盖掉其他颜色的涂鸦轨迹，这样覆盖的时候，底图img也会同步进行覆盖，此时在橡皮擦涂鸦功能完成以后，再一次重新渲染底图。

###### 思路
1. 打开弹框  
``` 
this.showImg = true;  
```

2. 初始化canvas,即初始化fabric  
``` 
this.initCanvas()  
```

3. 清除画布  
``` 
this.canvas.clear()  
```

4. 压缩图片  
``` 
this.pressImg()  
```

5. 计算图片大小，位置  
``` 
this.resetImg()   
```

6. 导入图片到画布  
``` 
this.imgToCanvas();   
```

7. 根据img计算出的precent,得出canvas的情况  
```
this.mathPrecent()   
```

8. 设置画笔大小,设置画笔颜色      
```
this.canvas.freeDrawingBrush.color = this.textColor;  
this.canvas.freeDrawingBrush.width = this.brushWidth;  
```

9. 监听 canvas 事件，绑定画板事件  
```
canvas.on("mouse:down", (options) => {}）    
canvas.on("mouse:up", (options) => {}）   
canvas.on("mouse:move", (options) => {}）  
```

10. drawing()事件，在mouse:move里面进行监听  
``` 
利用switch    beack，把画画的随意画，画矩形，写文字逻辑都写入。  
```

11. 删除操作，利用插件暴露api进行删除。参考上方给出文档  
```
this.canvas.remove(e.target);  
this.canvas.renderAll();  
```

12. 批改完成后，把图片转化为base64的数据，并发送数据与后端进行交互  

比较复杂的逻辑是： 
1. 图片渲染时，如果图片的宽高超过了屏幕可视区域的宽高，则需要对图片，以及图片渲染的canvas  按照浏览器的宽高比，进行同等比例的缩放。
2. 图片在涂鸦的过程中，需要对图片进行放大缩小，以及拖拽。在网上找到的涂鸦工具，全部都是固定的宽高，底图，无法操作画板的大小，比例。这个难题解决了很久。
第一期开发功能，在保证公司分配任务，利用剩余事件开发，思考，耗时一个月。

