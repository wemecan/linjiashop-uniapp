<template>
	<view class="wrap">
		<u-cell-group>
			<u-cell-item icon="bag" title="订单编号" :arrow="false" :value="orderSn"></u-cell-item>
			<u-cell-item icon="rmb" title="应付金额" :arrow="false" :value="totalPriceFmt"></u-cell-item>
		</u-cell-group>
		<u-radio-group v-model="payType" style="width:100%;">
			<view class="item" v-for="(res, index) in payWayList" :key="res.name">
				<u-row>
					<u-col :span="11">
						<view class="top">
							<view class="name">
								<u-image mode="widthFix" width="200rpx" :src="res.img"></u-image>
							</view>
						</view>
					</u-col>
					<u-col :span="1">
						<u-radio :name="res.name"></u-radio>
					</u-col>
				</u-row>
			</view>
		</u-radio-group>

		<u-button type="primary" @click="submit">立即支付</u-button>

	</view>
</template>

<script>
	// #ifdef H5
	import wx from '@/common/weixin-js-sdk'
	//#endif
	export default {
		data() {
			return {
				orderSn: '',
				totalPrice: '',
				payType: 'wxpay',
				payWayList: [{
						name: 'wxpay',
						text: '微信支付',
						img: '/static/img/wxpay.png'
					}

				]
			}
		},
		computed: {
			totalPriceFmt() {
				return '¥' + (this.totalPrice / 100).toFixed(2)
			}
		},
		onLoad(option) {
			this.orderSn = option.orderSn
			this.totalPrice = option.totalPrice
			//使用微信访问本系统的时候获取微信openid，否则不获取
			const userAgent = window.navigator.userAgent.toLowerCase()
			if (userAgent.indexOf('micromessenger') <= -1) {
				this.payWayList.push({
					name: 'alipay',
					text: '支付宝',
					img: '/static/img/alipay.png'
				})
			}
			this.init()


		},
		methods: {
			init() {
				this.$u.get('pay/queryResult/' + this.orderSn).then(res => {
					//如果当前订单已经支付成功跳转到订单详情页
					if (res == true) {
						this.$u.route({
							url: '/pages/order/detail',
							params: {
								orderSn: orderSn
							}
						})
					}
				})
			},
			submit() {
				console.log('payType', this.payType);
				// #ifdef H5
				if ('wxpay' === this.payType) {
					this.$u.post('pay/wx/prepare?orderSn=' + this.orderSn).then(res => {

						this.wxPayH5({
								appId: res.appId,
								nonceStr: res.nonceStr,
								package: res.packageValue,
								paySign: res.paySign,
								signType: res.signType,
								timeStamp: res.timeStamp
							}

						)
					})
				}
				// #endif
				
				// #ifndef H5
				if ('wxpay' === this.payType) {
					
					this.$u.post('pay/wx/prepare?orderSn=' + this.orderSn).then(res => {
						uni.requestPayment({
							provider: 'wxpay',
							timeStamp: res.timeStamp, //时间戳
							nonceStr:res.nonceStr, //随机字符串
							package: res.packageValue, //统一下单接口返回的 prepay_id 参数值
							signType:res.signType,
							paySign: res.paySign, //签名内容
							success: (e) => { 
								me.queryPayResult()
							},
							fail: (e) => {
								console.log("fail", e);
								if(e.errMsg.indexOf('denied')>-1){ 
									uni.showModal({
										content: '该小程序暂未开通支付功能',
										showCancel: false
									})
								}else{
									uni.showModal({
										content: "APP支付失败,原因为: " + e.errMsg,
										showCancel: false
									})
								}
								this.$u.route({
									url: '/pages/order/detail',
									params: {
										orderSn: this.orderSn
									}
								})
							}
						})
					});
				}
				// #endif
			},
			wxPayH5: function(data) {
				//获取后台传入的数据
				let appId = data.appId
				let timestamp = data.timeStamp
				let nonceStr = data.nonceStr
				let signature = data.signature
				let packageValue = data.package
				let paySign = data.paySign
				let signType = data.signType
				const that = this
				//下面要发起微信支付了
				wx.config({
					debug: false, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。
					appId: appId, // 必填，公众号的唯一标识
					timestamp: timestamp, // 必填，生成签名的时间戳
					nonceStr: nonceStr, // 必填，生成签名的随机串
					signature: signature, // 必填，签名，见附录1
					jsApiList: ['chooseWXPay'] // 必填，需要使用的JS接口列表，所有JS接口列表见附录2
				})
				let me = this
				wx.ready(function() {
					wx.chooseWXPay({
						timestamp: timestamp, // 支付签名时间戳，注意微信jssdk中的所有使用timestamp字段均为小写。但最新版的支付后台生成签名使用的timeStamp字段名需大写其中的S字符
						nonceStr: nonceStr, // 支付签名随机串，不长于 32 位
						package: packageValue, // 统一支付接口返回的prepay_id参数值，提交格式如：prepay_id=***）
						signType: signType, // 签名方式，默认为'SHA1'，使用新版支付需传入'MD5'
						paySign: paySign, // 支付签名
						success: function(res) {
							// 支付成功后的回调函数
							me.queryPayResult()

						},
						fail: function(res) { 
							//失败回调函数
							this.$u.toast('支付失败')
							this.$u.route({
								url: '/pages/order/detail',
								params: {
									orderSn: this.orderSn
								}
							})
							
						}
					})
				})
			},
			queryPayResult() {
				this.$u.get('pay/queryResult/' + this.orderSn).then(res => {
					this.$u.route({
						url: '/pages/order/detail',
						params: {
							orderSn: this.orderSn
						}
					})
				})
			},
		}
	}
</script>


<style lang="scss" scoped>
	.wrap {
		padding: 0rpx 30rpx;

		.item {
			width: 100%;
			padding: 20rpx 10rpx;

		}



	}
</style>
