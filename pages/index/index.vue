<template>
    <view >
      <view class="topTitle">已发现 {{deviceNum}} 个蓝牙设备</view>
      <scroll-view scroll-y scroll-with-animation  class="deviceListWrap">
        <uni-collapse  @change="onChange" accordion="true">
            <uni-collapse-item :title="item.name" v-for="(item,index) in deviceList" :key="index">
              <view slot="title">{{item.name}}</view>
              <view>信号强度：{{item.RSSI}}dBm({{(100 + item.RSSI)}}%)</view> 
              <view>UUID：{{item.deviceId}}</view> 
              <view class='concatBtn'><button size="mini" type="primary" @click="connetBlue(0,index)" >连接设备</button></view>  
            </uni-collapse-item>
        </uni-collapse>
      </scroll-view> 
      <view class="connectedStatus">
		  
		<button size="defult" type="primary" @click="initBluetoothModule" style="margin: 20upx 0;">开始扫描</button>
		<button size="defult" type="defult" @click="closeBluetoothAdapter" style="margin: 20upx 0;">停止扫描</button>
        <view v-if="connected" class="nowConcat">已连接到：{{connectedName}}</view> 
        <view  v-if="prevConnected">
          <view  class="npConcat">上次连接：{{pDeviceInfo.name}}</view> 
          <button size="defult" type="defult" @click="connetBlue(1,-1)">直接连接</button>
        </view> 
		 <view v-if="connected" class="nowConcat">
			<button size="defult" type="defult" @click="getBLEDeviceServices" style="margin: 20upx 0;">测试打印</button>
		</view> 
      </view>  
    </view>
</template>

<script>
	import uniCollapse from "@/plugs/uni-ui/uni-collapse/uni-collapse.vue"
	import uniCollapseItem from "@/plugs/uni-ui/uni-collapse-item/uni-collapse-item.vue"
	export default {
		components: {uniCollapse,uniCollapseItem},
		data() {
			return {
				deviceList:[],
				deviceNum:0,
				activeNames:0,
				connected:false,
				connectedName:'',
				prevConnected:false,
				pDeviceInfo:{},
				deviceId:'',
				serviceId:'',
				characteristics:'',
			};
		},
		onLoad() {
			//初始蓝牙模块
			this.pDeviceInfo = uni.getStorageSync('deviceInfo');
			if(!!this.pDeviceInfo){
				this.prevConnected = true;
			}
			//this.initBluetoothModule()
		},
		methods:{
			onChange(event){
				this.activeNames =event.detail; 
			},
			initBluetoothModule(){
				//初始蓝牙模块
				uni.openBluetoothAdapter({
				  success:res=> {
					this.searchBlueList();
				  },
				  fail:err=>{
					console.log(err)
				  }
				})
			},
			searchBlueList(){
				//开启蓝牙搜索
				uni.startBluetoothDevicesDiscovery({
					success:res=> {
						setTimeout(()=>{
							this.getBlueList();
							  uni.showToast({
								title: '开启成功',
								icon: 'success',
								duration: 1000
							});
						},1000);
					},
				})
			},
			getBlueList(){
				//获取搜索列表
				uni.getBluetoothDevices({
				  success:res=> {
					let data = res.devices
					let tempList=[];
					data.map(device=>{
					  if(!!device.localName){
						tempList.push(device)
					  }
					});
					this.deviceNum = tempList.length;
					this.deviceList=tempList;
					this.listenBluetooth();
				  }
				});
			},
			listenBluetooth(){
				let tempList =this.deviceList;
				//监听蓝牙搜索
				uni.onBluetoothDeviceFound((res) => {
				  let flag = false;
				  res.devices.forEach(device => {
					if(!!device.localName){
					  tempList.push(device)
					  flag =true;
					}
				  })
				  if(flag){
					this.deviceList=tempList;
					this.deviceNum = this.deviceList.length;
				  }
			   })
			},
			connetBlue(type,index){
				let deviceIndex = index;
				let deviceInfo = this.deviceList[deviceIndex];
				if(this.prevConnected && type == 1){
				  deviceInfo = this.pDeviceInfo;
				} 
				let dId = deviceInfo.deviceId;
				uni.showLoading({
				  title: '正在连接...', //提示的内容,
				  mask: true
				});
				//连接蓝牙
				uni.createBLEConnection({
				  deviceId: dId,//设备id
				  success: res=> {
					uni.hideLoading();	
					if(res.errCode == 0){
					  this.connected = true;
					  this.connectedName=deviceInfo.name;
					  uni.setStorageSync('deviceInfo',deviceInfo);
					  this.deviceId=dId;
					  uni.showToast({
						title: '连接成功',
						icon: 'success',
						duration: 1000
					});
					}
					uni.stopBluetoothDevicesDiscovery({
					  success: res => {
						console.log('连接蓝牙成功之后关闭蓝牙搜索');
					  }
					})
				  },
				  fail:err=>{
					uni.showToast({
						title: '连接失败！', 
						icon: 'none', 
						duration: 2000
					  });
				  }
				})
			},
			getBLEDeviceServices(){
				//获取服务
				uni.showLoading({
				  title: '正在打印...', //提示的内容,
				  mask: true
				});
				let deviceId = this.deviceId;
				uni.getBLEDeviceServices({
					deviceId,
					success: (res) => {	
					  for (let i = 0; i < res.services.length; i++) {
						if (res.services[i].isPrimary) {
						  this.getBLEDeviceCharacteristics(deviceId, res.services[i].uuid);
						}
					  }
					},
					fail: (res) => {
					  uni.hideLoading();
					  console.log("获取蓝牙服务失败：" + JSON.stringify(res))
					}
				})
			},
			//获取单个服务的特征值(characteristic)
			getBLEDeviceCharacteristics(deviceId, serviceId){
				if(!!this.characteristics && !!this.serviceId){
					this.PrintStr();
					return;
				}
				uni.getBLEDeviceCharacteristics({
					deviceId,
					serviceId,
					success: (res) => {
						uni.hideLoading();
						for (let i = 0; i < res.characteristics.length; i++) {
							let item = res.characteristics[i];
							if (item.properties.write && !this.serviceId) {
								this.serviceId = serviceId;
								this.characteristics = item.uuid;
								this.PrintStr();
							}
						}
					},
					fail(res) {
					uni.hideLoading();
					console.error('获取特征值失败：', res)
					}
				})
			},
			PrintStr() {
				const buf = Buffer.from("this is test content!");
				var bufferstr = buf.buffer; 
				uni.writeBLECharacteristicValue({
					deviceId: this.deviceId,
					serviceId: this.serviceId,
					characteristicId: this.characteristics,
					value: bufferstr,
					success: function (res) {
						 uni.hideLoading();
					},
					fail: function (res) {
						console.log("数据发送失败:" + JSON.stringify(res))
					},
					complete: function (res) {
						
					},
				})
			},
			closeBluetoothAdapter(){
				uni.closeBluetoothAdapter({
				  success: res => {
					console.log('关闭蓝牙适配器');
				  }
				});
			},
		},
		onUnload() {
		  this.closeBluetoothAdapter();
		}
	}
</script>

<style >
.topTitle{font-size: 15px;padding: 5px;color: #999;text-align: center;}
.deviceListWrap{height: calc(100vh - 160rpx); }
.connectedStatus{position: fixed;bottom: 30rpx;left: 15%;width: 70%;text-align: center;border-top: 1px solid #eee;}
.concatBtn{text-align: center;margin: 30rpx 0;}
.nowConcat {margin: 5px 0;color: #333;}
.npConcat{margin: 5px 0;color: #999;}
</style>
