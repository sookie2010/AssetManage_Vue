<!-- 资产信息搜索结果 -->
<template>
<div>
<x-table :cell-bordered="false" >
	<thead>
		<tr style="background-color: #F7F7F7">
			<th style="width:22%">编码</th>
			<th style="width:22%">名称</th>
			<th style="width:22%">类别</th>
			<th style="width:16%">数量</th>
			<th style="width:18%">保管人</th>
		</tr>
	</thead>
</x-table>
<div class="table-body-container" >
<x-table :cell-bordered="false" >
	<tbody>
		<!-- TODO 使用 scroller InlineLoading 组件实现滚动加载 -->
		<tr v-for="(item,index) in zcList" :key="item.uuid" v-on:click="trClick(index)"
			v-bind:class="{selected : item.isSelected}">
			<td style="width:22%">{{item.zcid}}</td>
			<td style="width:22%">{{item.mingch}}</td>
			<td style="width:22%">{{item.lbie}}</td>
			<td style="width:16%">{{item.shul}}</td>
			<td style="width:18%">{{item.bgr.realname}}</td>
		</tr>
	</tbody>
</x-table>
</div>
<divider v-if="showTip">没有符合条件的数据</divider>
<div class="btn-container">
	<flexbox>
	<flexbox-item><x-button @click.native="addToList" type="primary">添加到清单</x-button></flexbox-item>
	<flexbox-item><x-button @click.native="showList" type="default">查看清单</x-button></flexbox-item>
	</flexbox>
</div>
<!-- 详细信息dialog -->
<x-dialog hide-on-blur :show.sync="showDialog" class="detail-dialog" @on-hide="dialogHideCallback">
	<div class="detail-panel">
		<group>
			<cell title="资产名称" primary="content" :value="selectIndex!==null?zcList[selectIndex].mingch:null"></cell>
			<cell-form-preview :list="datailList"></cell-form-preview>
			<cell title="相关照片" primary="content" :value="!imageList || !imageList.length ? '无' : null">
				<img v-for="(imgPath,index) in imageList" v-bind:key="index" width="100" class="previewer-img"
					@click="enlargePhoto(index)" :src="$store.state.readPhotoUrl+imgPath"/>
				<previewer :list="previewerList" ref="previewer" v-if="imageList.length" 
					:options="previewerOptions" @on-close="previewerIsOpen=false"></previewer>
			</cell>
		</group>
	</div>
	<div class="btn-container">
		<x-button @click.native="selectZc" :type="selectIndex===null||!zcList[selectIndex].isSelected?'primary':'warn'">
			{{selectIndex===null||!zcList[selectIndex].isSelected?'选中':'取消选中'}}
		</x-button>
	</div>
</x-dialog>
</div>
</template>

<script>
import { XTable,XButton,XDialog,CellFormPreview, Flexbox, FlexboxItem,Previewer,
		Group, Cell,Divider,TransferDomDirective as TransferDom } from 'vux'
//详情dialog当中包含的字段
const datailColumns = {ggxh:"规格",shul:"数量",ppcj:"厂家",gysDcxm:"供应商"};

export default {
	name : "result",
	data () {
		return {
			operate : localStorage.getItem("operate"),
			zcList : [],
			datailList :[],
			selectIndex : null, //选中行的索引
			showDialog : false, //dialog是否显示
			imageList : [], //dialog当中的图片列表
			showTip : false,
			previewerOptions: { //构建从缩略图放大展开的动画效果
				getThumbBoundsFn (index) {
				// 找到缩略图的元素
				let thumbnail = document.querySelectorAll('.previewer-img')[index]
				// 获取当前视窗的Y轴坐标(向下滚动了多少)
				let pageYScroll = window.pageYOffset || document.documentElement.scrollTop
				// 获取该缩略图的定位信息
				let rect = thumbnail.getBoundingClientRect()
				return {x: rect.left, y: rect.top + pageYScroll, w: rect.width}
				}
			},
			previewerIsOpen : false //图片放大预览是否打开
		};
	},
	computed : {
		//提供给预览视图的图片列表
		previewerList () {
			return this.imageList.map((imgPath) => {
				return {src : this.$store.state.readPhotoUrl + imgPath};
			});
		}
	},
	created () {
		this.$store.commit("setHeaderConf",{hasbackbtn : true,title : "查询结果"});
		var _this = this;
		const params = {
			zcid : this.$route.query.zcid,
			mingch : this.$route.query.name,
			lbie : this.$route.query.type
		};
		switch(this.operate) {
		case "1" : //出库
			//查询所有属于MA名下的数据(也就是未出库的)
			params.role = "MA";
			break;
		case "2" : //流转
			//查询所有属于MK名下的数据
			params.role = "MK";
			break;
		case "3" : //TODO 回收
		}
		//查询资产数据
		this.$http.get(this.$store.state.apiUrl + "zichan/list",{params})
		.then((response) => {
			_this.zcList = response.data;
			if(!_this.zcList.length) {
				_this.showTip = true;
			}
		});
	},
	directives: { TransferDom },
	components : { XTable,XButton,XDialog,CellFormPreview,Previewer,
			Flexbox,FlexboxItem,Group,Cell,Divider },
	methods : {
		/**
		* 数据表格行点击事件函数
		* @argument index 数据行的索引(从0开始)
		*/
		trClick (index) {
			this.selectIndex = index;
			this.showDialog = true;
			var datailList = [];
			for(let name in this.zcList[index]) {
				if(name in datailColumns) {
					datailList.push({
						label : datailColumns[name],
						value : this.zcList[index][name]
					});
				}
			}
			this.datailList = datailList;
			var _this = this;
			//获取资产流转的最后一张照片
			this.$http.post(this.$store.state.apiUrl + "zichan/findLastPhoto", {zcid : this.zcList[index].zcid})
			.then(function(response){
				_this.imageList = response.data;
			});
		},
		/**
		* 选中/取消选中当前选择的资产数据
		*/
		selectZc () {
			this.showDialog = false;
			var _this = this;
			var isSelected = !this.zcList[this.selectIndex].isSelected;
			setTimeout(()=>{
				_this.zcList.splice(_this.selectIndex, 1,
					Object.assign(_this.zcList[_this.selectIndex], {isSelected})
				);
			},200);
		},
		/**
		* "添加到清单"按钮点击事件函数
		* 将选中的数据添加到清单
		*/
		addToList () {
			var selectedIds = this.zcList.filter(function(item){
				return item.isSelected;
			}).map(function(item){
				return item.uuid;
			});
			if(selectedIds.length == 0) {
				this.$vux.toast.text('请选择加入到清单的资产', 'middle');
				return;
			}
			//由于可以多次添加, 需要获取到上次添加的列表
			var oldList = localStorage.getItem("selectedIds");
			if(!oldList) {
				//第一次添加
				localStorage.setItem("selectedIds", JSON.stringify(selectedIds));
			} else {
				//不是第一次添加
				let currentList = JSON.parse(oldList);
				for(let index in selectedIds) {
					if(currentList.findIndex(function(item){return item===selectedIds[index]}) == -1) {
						//现有列表中没有该ID
						currentList.push(selectedIds[index]);
					}
				}
				localStorage.setItem("selectedIds", JSON.stringify(currentList));
			}
			this.$vux.toast.text('添加成功', 'middle')
		},
		/**
		* "查看清单"按钮点击事件函数
		*/
		showList () {
			var selectedIds = localStorage.getItem("selectedIds");
			if(!selectedIds || !JSON.parse(selectedIds).length) {
				this.$vux.alert.show({ title: '提示',content: '清单中没有任何数据' });
				return;
			}
			this.$router.push("/zc/list");
		},
		/**
		 * 放大查看图片
		 * @argument index 图片的索引
		 */
		enlargePhoto (index) {
			this.$refs.previewer.show(index);
			this.previewerIsOpen = true;
		},
		/**
		 * dialog被隐藏时执行的回调函数
		 */
		dialogHideCallback () {
			if(this.$refs.previewer && this.previewerIsOpen) {
				this.$refs.previewer.close();
				this.imageList = [];
			}
		}
	}
}
</script>
<style scoped>
tr.selected {
	background-color: rgb(238,248,172);
}
.detail-dialog .detail-panel{
	max-height: calc(70vh);
	overflow:scroll;
}
.table-body-container {
	max-height: calc(100% - 7.1em);
}
</style>
