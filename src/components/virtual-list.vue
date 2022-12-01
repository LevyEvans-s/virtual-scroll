<template>
  <div class="viewport" ref="viewport" @scroll="scrollFn">
    <!-- 滚动条，高度等于所有Item的高度之和 -->
    <div class="scrollBar" ref="scrollBar"></div>
    <!-- 渲染区，需动态计算偏移 -->
    <div class="scroll-list" :style="{transform:`translate3d(0,${offset}px,0)`}">
      <div v-for="(item,index) in visibleData" :key="item.id" :vid="item.index" ref="items">
        <slot :item="item"></slot>
      </div>
    </div>
  </div>
</template>
<script>
import throttle from 'lodash/throttle'
export default {
  props: {
    items: Array,
    size: Number,
    remain: Number,
    variable: Boolean
  },
  data() {
    return {
      start: 0,
      end: null,
      offset: 0
    };
  },
  created(){
    // 节流并且第一次不触发
    this.scrollFn=throttle(this.handleScroll,200,{leading:false})
  },
  computed: {
    formatData(){
      return this.items.map((item,index)=>({...item,index}))
    },  
    visibleData() { // 可见的数据，根据start和end自动去items中截取渲染
      let start = this.start - this.prevCount;
      let end = this.end + this.nextCount;
      return this.formatData.slice(start, end);
    },
    prevCount() { // 前面预留几个
      return Math.min(this.start, this.remain);
    },
    nextCount() { // 后面预留几个
      return Math.min(this.items.length - this.end, this.remain);
    }
  },
  mounted() {
    // 1.设置viewPrort的高度
    this.$refs.viewport.style.height = this.remain * this.size + "px";
    // 2.设置滚动条高度
    this.$refs.scrollBar.style.height = this.items.length * this.size + "px";
    // 3.计算显示范围
    this.end = this.start + this.remain;

    // 如果加载完毕 需要先缓存每一项高度
    // 先记录好，等一会滚动的时候 去渲染页面时获取真实dom的高度 来更新缓存的内容
    if (this.variable) {
      this.initPosition();
    }
  },
  updated(){
      // 页面渲染完成后 需要根据当前展示的数据更新缓存区的内容
      // 获取真实元素的位置 更新top和bottom;
     this.$nextTick(()=>{
      // 根据当前显示的 更新缓存中的height、bottom、top，最终更新滚动条高度
        let nodes = this.$refs.items;
        if(!(nodes && nodes.length >0)){
            return 
        }
        nodes.forEach(node=>{ 
            let rect = node.getBoundingClientRect();
            let height = rect.height; // 真实的dom
            let index = +node.getAttribute('vid');
            let oldHeight = this.positions[index].height;
            let val = oldHeight - height; // 计算当前的高度是否和之前有变化
            // console.log(val);
            if(val){ // 如果缓存的和真实的高度一样 就不用任何操作
                // 先更新自己
                this.positions[index].bottom = this.positions[index].bottom - val; // 元素高度变了 自然bottom也会变
                this.positions[index].height = height;
                // 所有元素都得变
                for(let i = index+1;i<this.positions.length;i++){
                    this.positions[i].top = this.positions[i-1].bottom;
                    this.positions[i].bottom = this.positions[i].bottom - val;
                }
            }
        })
        // 动态计算滚动条的高度：只要更新过 就会算出滚动条的最新高度
        this.$refs.scrollBar.style.height = this.positions[this.positions.length-1].bottom +'px';
        // this.offset = this.positions[this.start - this.prevCount]? this.positions[this.start - this.prevCount].top : 0;
     });
  },
  methods: {
    initPosition() {
      // 初始化位置
      this.positions = this.items.map((item, index) => ({
        index,
        height: this.size,
        top: index * this.size,
        bottom: (index + 1) * this.size
      }));
    },
    getStartIndex(value) {
      let positions = this.positions;
      let start = 0;
      let end = this.positions.length;
      let temp = null;
      while (start < end) {
        let middleIndex = parseInt((start + end) / 2);
        let middleValue = this.positions[middleIndex].bottom;
        if (value == middleValue) {
          return middleIndex + 1;
        } else if (middleValue < value) {
           start = middleIndex + 1;
        } else if (middleValue > value) {
          if (temp == null || temp > middleIndex) {
            temp = middleIndex;
          }
          end = middleIndex - 1;
        }
      }
      return temp;
    },
    handleScroll() {
      let scrollTop = this.$refs.viewport.scrollTop;
      if (this.variable) {// 使用二分查找找到对应记录
        // 算出开始的位置
        this.start = this.getStartIndex(scrollTop);
        this.end = this.start + this.remain;
        this.offset = this.positions[this.start - this.prevCount]? this.positions[this.start - this.prevCount].top : 0;
        // 算出结尾位置
        // 设置偏移量
      } else {
        // 计算开始
        this.start = Math.floor(scrollTop / this.size);
        // 计算结束
        this.end = this.start + this.remain;
        // 计算偏移量
        this.offset =
          scrollTop - (scrollTop % this.size) - this.prevCount * this.size; // 如果有预留渲染 应该把这个位置再向上移动预留的总高度
      }
    }
  },

};
</script>
<style lang="stylus">
.viewport {
  overflow-y: scroll;
  position: relative;
}

.scroll-list {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
}
</style>