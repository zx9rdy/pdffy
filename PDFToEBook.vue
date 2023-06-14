<template>

  <div class="pdfList"></div>
  <div class="bookBox align-center direction-column">
    <div id="turnpage" class="turnpage">
      <div v-for="(item, index) in pdfPicturePath" :key="index"
        :style="` background:url(${item.imgurl}) no-repeat 100% 100%; background-size:100% 100%;height:100px;`">
        <!-- <div class="pagenum">第{{ item.index }}页</div> -->
      </div>
    </div>
    <div class="flex align-center" style="margin-top: 25px; padding-bottom: 30px">
      <p>
        本文档共<span style="color: #0d84ff">{{ bookPages }}</span>页，您想直接查看第
      </p>
      <el-input v-model="toPage" placeholder="" min="1" :max="bookPages" class="toPage" type="number"></el-input>
      <span>页</span>
      <el-button type="primary" size="mini" @click="toPageNum" style="margin-left: 10px">跳转</el-button>
      <el-button type="primary" size="mini" @click="scalePDF" style="margin-left: 10px">放大</el-button>
    </div>
  </div>
  <div class="dialogBox">
    <el-dialog v-model="dialogVisible" width="100%" :align-center="true" custom-class="aaaa" :show-close="false"
      :close-on-click-modal="false" :close-on-press-escape="false">
      <div class="scaleBox">
        <span class="el-icon-circle-close closeBtn" @click="closeScale"></span>
        <div class="booxBox2">
          <div id="biggerEBook" class="biggerEBook">
            <div v-for="(item, index) in pdfPicturePath" :key="index"
              :style="` background:url(${item.imgurl}) no-repeat 100% 100%; background-size:100% 100%;height:100px;`">
              <!-- <div class="pagenum">第{{ item.index }}页</div> -->
            </div>
          </div>
        </div>
      </div>
    </el-dialog>
  </div>
</template>

<script >

import { defineComponent } from "vue";

import * as pdfjs from "pdfjs-dist";
import * as pdfjsWorker from "pdfjs-dist/build/pdf.worker.entry";

pdfjs.GlobalWorkerOptions.workerSrc = pdfjsWorker;

import $ from "jquery";
import turn from "../../../public/turnJS/turn.js";

import { ElLoading } from "element-plus";

const loadingInstance = ElLoading.service({
        fullscreen: true,
        text: "资源请求中，请稍候",
      });
export default defineComponent({
  name: "PDFToEBook",
  // 注册你的组件
  components: {},

  data() {
    return {
      dialogVisible: false,
      toPage: 1,
      bookPages: 6,
      props: { multiple: true },
      value: "",
      page: 1,
      rightBorderImg: require("../../assets/images/right-border.png"),
      leftBorderImg: require("../../assets/images/left-border.png"),
      imgFiles: [],
      pdfName: "",
      pdfPicturePath: [],
      pdfFile: "static/docs.pdf",
      alreadyDraw: false,
    };
  },
  methods: {

    getPDFBlob() {

      let netFilePath='netFile/'+this.$route.query.filepath //'netFile/PDF/M/2023/2/14/M-2023-02-14-1.pdf'
      // console.log("文件地址是,",netFilePath)
      let that = this
      this.$http.getFile(netFilePath, {}).then(res => {
        // console.log('二进制', res.data)
        const blob = new Blob([res.data], { type: "application/pdf" });
        // console.log("1111",blob)
        // const url = URL.createObjectURL(blob)
        // const a = document.createElement("a");
        // a.href = url;
        // a.target = '_blank'
        // a.click();

        let reader = new FileReader(); //实例化文件读取对象
        reader.readAsDataURL(res.data); //将文件读取为 DataURL,也就是base64编码
        reader.onload = (e) => { //文件读取成功完成时触发
          let dataURL = e.target.result; //获得文件base64编码
          let tempStr = dataURL.substr(dataURL.indexOf('base64,') + 7)
          that.showPdf(tempStr)
        }
      })

    },
    async showPdf(dataURL) {
      // const loadingInstance = ElLoading.service({
      //   fullscreen: true,
      //   text: "转换中，请稍候",
      // });

      let base64 = dataURL; //获得bas464编码
      let decodedBase64 = atob(base64); //使用浏览器自带的方法解码
      let pdf = await pdfjs.getDocument({ data: decodedBase64 }); //返回一个pdf对象  http://www.sinopecgroup.com/group/Resource/Pdf/20220923xcc.pdf  http://www.gov.cn/zhengce/pdfFile/2021_PDF.pdf
      let pages = pdf.numPages; //声明一个pages变量等于当前pdf文件的页数
      for (let i = 1; i <= pages; i++) {
        //循环页数
        let canvas = document.createElement("canvas");
        let page = await pdf.getPage(i); //调用getPage方法传入当前循环的页数,返回一个page对象
        let scale = 1; //缩放倍数，1表示原始大小
        let viewport = page.getViewport(scale);
        let context = canvas.getContext("2d"); //创建绘制canvas的对象
        canvas.height = viewport.height; //定义canvas高和宽
        canvas.width = viewport.width;
        let renderContext = {
          canvasContext: context,
          viewport: viewport,
        };
        await page.render(renderContext);
        canvas.className = "canvas"; //给canvas节点定义一个class名,这里我取名为canvas
        let imgObj = canvas.toDataURL("image/png");
        let pdfImg = {
          imgurl: imgObj,
          index: i,
        };
        this.pdfPicturePath.push(pdfImg);
      }
      await this.drawPdf("turnpage");
      loadingInstance.close();
    },

    scalePDF() {
      this.dialogVisible = true;
      if (!this.alreadyDraw) {
        this.$nextTick(() => {
          this.drawPdf("biggerEBook");
        });
      }
    },
    drawPdf(containerName) {
      const that = this;
      console.log($("#" + containerName), this.pdfPicturePath);
      this.$nextTick(() => {
        // this.bookPages = $("#turnpage").turn("pages");
        $("#" + containerName).turn({
          acceleration: true, //启用硬件加速,移动端有效
          display: "double", //显示：single=单页，double=双页，默认双页
          duration: 800, // 翻页撒开鼠标，页面的延迟
          page: 1, // 默认显示第几页
          gradients: true, // 折叠处的光泽渐变，主要体现翻页的立体感、真实感
          autoCenter: true, //
          turnCorners: "bl,br", // 设置可翻页的页角(都试过了，乱写 4个角都能出发卷起)， bl,br   tl,tr   bl,tr

          when: {
            turning: function (e, page, view) {
              const pagesTotalNum = $("#" + containerName).turn("pages");
              if (page != pagesTotalNum && page != 1) {
                $(".page").children(".pageShadow").remove();
                const leftHtmlStr = `	<div class="pageShadow" style='z-index:999;position: absolute;top: 0;right: 0; width: 15px;height: 100%;'>
			        <img src=${that.rightBorderImg} style="width:100%;height:100%">
		        </div>`;
                const rightHtmlStr = `	<div class="pageShadow" style='z-index:999;position: absolute;top: 0;left: 0; width: 15px;height: 100%;'>
			        <img src=${that.leftBorderImg} style="width:100%;height:100%">
		        </div>`;
                $(".even.p" + view[0]).append(leftHtmlStr);
                $(".odd.p" + view[1]).append(rightHtmlStr);
              }
            },
          },
        });

        $("#" + containerName).on("mousewheel DOMMouseScroll", function (e) {
          if (e.originalEvent.wheelDelta > 0) {
            $("#" + containerName).turn("previous");
          } else {
            $("#" + containerName).turn("next");
          }
        });
      });
      if (containerName == "biggerEBook") {
        this.alreadyDraw = true;
      }
    },
    closeScale() {
      this.dialogVisible = false;
    },
  },
  mounted() {
    // this.showPdf(test);
    this.getPDFBlob()
  },
});
</script>

<style  scoped>
.bookBox {
  width: 60%;
  height: 100%;
  display: flex;
  align-items: center;
  text-align: center;
  margin: 0 auto;
  padding-top: 30px;
}

#turnpage {
  width: 100%;
  height: 700px;
}

.pageShadow {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

.shadow {
  box-shadow: 0 0 20px #d2d2d2 !important;
}

.toPage {
  width: 50px;
}

.toPage>>>.el-input__inner {
  text-align: center;
  height: 25px !important;
  line-height: 25px !important;
  padding: 0 !important;
}

.scaleBox {
  width: 70%;
}

.closeBtn {
  position: fixed;
  top: 30px;
  right: 30px;
  font-size: 30px;
  font-weight: 500;
  color: #fff;
  z-index: 2222;
}

.closeBtn:hover {
  cursor: pointer;
  transform: rotate(180deg);
  transition: all 0.3s;
}

.booxBox2 {
  width: 100%;
  height: 800px;
  display: flex;
  align-items: center;
  text-align: center;
  margin: 0 auto;
}

.biggerEBook {
  width: 100%;
  height: 100%;
}

.dialogBox>>>.el-dialog {
  margin: unset !important;
  background: unset !important;
  box-shadow: unset !important;
}

.dialogBox>>>.el-dialog__body {
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>