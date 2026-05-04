<template>
	<view class="chat-container" :style="{ height: containerHeight }">
		<view class="status-bar"></view>
		
		<view class="connection-header">
			<view class="left-section" @tap="goBack">
				<text class="back-icon">〈</text>
				<view class="conn-status">
					<view class="status-dot" :class="{ active: isConnected }"></view>
					<text class="conn-text">{{ deviceName }}</text>
				</view>
			</view>
			<text class="page-title">实时对话</text>
		</view>

		<scroll-view 
			class="chat-window" 
			scroll-y 
			:scroll-into-view="lastMsgId" 
			scroll-with-animation
			@scroll="onChatScroll">
			<view v-for="(msg, index) in messages" :key="index" :id="'msg-' + index" 
				class="msg-row" :class="msg.type">
				<view class="msg-content">
					<text class="msg-text">{{ msg.text }}</text>
					<text class="msg-time">{{ msg.time }}</text>
				</view>
			</view>
			<view style="height: 120rpx;"></view>
		</scroll-view>

		<view class="floating-controls">
			<view class="float-btn red" @tap="clearMessages">✕</view>
			<view class="float-btn green" @tap="downloadLog">↓</view>
		</view>

		<view class="input-panel">
			<view class="input-row">
				<input class="main-input capsule-input" 
					:class="{ 'input-sending': autoRepeat }"
					v-model="inputText" 
					:disabled="autoRepeat"
					placeholder="输入指令..." 
					confirm-type="send" 
					@confirm="sendData(false)" />
				
				<view v-if="!autoRepeat" class="send-btn-circle" @tap="sendData(false)">↑</view>
				<view v-else class="send-btn-circle stop-btn" @tap="autoRepeat = false">■</view>
			</view>
			
			<view class="config-row">
				<view class="checkbox-item" @tap="isHex = !isHex">
					<view class="custom-radio" :class="{ 'radio-active': isHex }"></view>
					<text :class="{'text-blue': isHex}">HEX 模式</text>
				</view>
				<view class="checkbox-item" @tap="toggleAutoRepeat">
					<view class="custom-radio red-radio" :class="{ 'radio-active': autoRepeat }"></view>
					<text :class="{'text-danger': autoRepeat}">自动重发</text>
				</view>
				<view class="repeat-interval capsule-bg">
					<input type="number" v-model="interval" class="interval-input" />
					<text class="unit-text">ms</text>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			deviceName: '未连接',
			deviceId: '',
			protocol: '',
			serviceId: '',
			characteristicId: '',
			isConnected: false,
			inputText: '',
			isHex: false,
			autoRepeat: false,
			interval: 1000,
			messages: [], 
			lastMsgId: '',
			repeatTimer: null,
			isAtBottom: true,
			keyboardHeight: 0,
			sppReadTimer: null,
			
			receiveBuffer: '',     
			receiveTimer: null     
		}
	},
	computed: {
		containerHeight() {
			return `calc(100vh - ${this.keyboardHeight}px)`;
		}
	},
	onLoad(options) {
		if (options.deviceName) {
			this.deviceName = decodeURIComponent(options.deviceName);
			this.deviceId = options.deviceId;
			this.protocol = options.protocol;
			this.isConnected = true;
		}
		this.loadHistory();
		uni.onKeyboardHeightChange(this.handleKeyboard);
		if (this.protocol === 'BLE' && this.deviceId) {
			this.initBLEService();
		}
		this.listenBluetoothData();
		uni.$on('bluetoothDisconnected', () => this.handleInternalDisconnect());
		uni.$on('updateChat', () => this.loadHistory());
	},
	onShow() { this.loadHistory(); },
	onUnload() {
		this.stopRepeat();
		if (this.sppReadTimer) {
			clearInterval(this.sppReadTimer);
			this.sppReadTimer = null;
		}
		if (this.receiveTimer) clearTimeout(this.receiveTimer);
		uni.offKeyboardHeightChange(this.handleKeyboard);
		uni.$off(['updateChat', 'bluetoothDisconnected']);
	},
	watch: {
		autoRepeat(newVal) {
			if (newVal) this.startRepeat();
			else this.stopRepeat();
		},
		isHex(newVal) {
			// 切换模式时清空缓冲区，防止数据乱码错位
			this.receiveBuffer = '';
		}
	},
	methods: {
		// --- HEX 工具函数 ---
		stringToHex(str) {
			let hex = '';
			for (let i = 0; i < str.length; i++) {
				hex += str.charCodeAt(i).toString(16).padStart(2, '0') + ' ';
			}
			return hex.toUpperCase().trim();
		},
		hexToArrayBuffer(hexStr) {
			let cleanHex = hexStr.replace(/\s+/g, '');
			if (cleanHex.length % 2 !== 0) cleanHex = '0' + cleanHex;
			const buffer = new ArrayBuffer(cleanHex.length / 2);
			const dataView = new DataView(buffer);
			for (let i = 0; i < cleanHex.length; i += 2) {
				dataView.setUint8(i / 2, parseInt(cleanHex.substring(i, i + 2), 16));
			}
			return buffer;
		},
		arrayBufferToHex(buffer) {
			const hexArr = Array.prototype.map.call(
				new Uint8Array(buffer),
				bit => ('00' + bit.toString(16)).slice(-2)
			);
			return hexArr.join(' ').toUpperCase();
		},

		processReceivedData(newStr, rawBuffer = null) {
			if (this.receiveTimer) clearTimeout(this.receiveTimer);
			
			// 区分 HEX 模式与字符模式拼接
			if (this.isHex && rawBuffer) {
				this.receiveBuffer += (this.receiveBuffer ? ' ' : '') + this.arrayBufferToHex(rawBuffer);
			} else {
				this.receiveBuffer += newStr;
			}

			this.receiveTimer = setTimeout(() => {
				if (this.receiveBuffer.length > 0) {
					this.addMessage('receive', this.receiveBuffer, true);
					this.receiveBuffer = ''; 
				}
			}, 300); 
		},

		handleInternalDisconnect() {
			this.isConnected = false;
			this.deviceName = '连接已断开';
			this.stopRepeat();
			if (this.sppReadTimer) {
				clearInterval(this.sppReadTimer);
				this.sppReadTimer = null;
			}
			uni.showToast({ title: '蓝牙已断开', icon: 'none' });
		},

		loadHistory() {
			if (!this.deviceId) return;
			const history = uni.getStorageSync('chat_history_' + this.deviceId);
			if (history) {
				this.messages = history;
				if (this.isAtBottom) this.scrollToBottom();
			}
		},

		handleKeyboard(res) {
			this.keyboardHeight = res.height;
			if (res.height > 0) setTimeout(() => { this.scrollToBottom(); }, 100);
		},

		onChatScroll(e) {
			this.isAtBottom = (e.detail.scrollHeight - e.detail.scrollTop < 850);
		},

		scrollToBottom() {
			this.$nextTick(() => {
				if (this.messages.length > 0) {
					this.lastMsgId = ''; 
					this.$nextTick(() => {
						this.lastMsgId = 'msg-' + (this.messages.length - 1);
					});
				}
			});
		},

		addMessage(type, text, isAuto = false) {
			if (!text || text.trim() === '') return;
			const now = new Date();
			const timeStr = `${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}:${now.getSeconds().toString().padStart(2, '0')}`;
			
			let history = uni.getStorageSync('chat_history_' + this.deviceId) || [];
			const cleanText = text.replace(/\r\n/g, ' ').replace(/\n/g, ' ');
			history.push({ type, text: cleanText, time: timeStr });
			
			// 【优化】限制最大保存条数，防止 UI 渲染和本地存储卡死
			if (history.length > 200) {
				history = history.slice(-200);
			}

			this.messages = history;
			uni.setStorageSync('chat_history_' + this.deviceId, history);
			if (!isAuto || this.isAtBottom) this.scrollToBottom();
		},

		sendData(isAuto = false) {
			if (!this.isConnected) {
				this.stopRepeat();
				return uni.showToast({ title: '未连接', icon: 'none' });
			}
			if (!this.inputText) {
				if (this.autoRepeat) this.autoRepeat = false;
				return;
			}
			
			const currentText = this.inputText;
			const app = getApp();

			if (this.protocol === 'SPP' || (app.globalData && app.globalData.sppSocket)) {
				this.sendBySPP(currentText, isAuto);
			} else {
				this.sendByBLE(currentText, isAuto);
			}
		},

		sendBySPP(text, isAuto) {
			try {
				const app = getApp();
				let outStream = app.globalData.sppSocket.getOutputStream();
				plus.android.importClass(outStream);
				
				// 兼容 HEX 模式发送
				if (this.isHex) {
					let buffer = this.hexToArrayBuffer(text);
					let bytesArray = new Uint8Array(buffer);
					// SPP 发送 byte 数组需借用反射转换
					let JavaArray = plus.android.invoke("java.lang.reflect.Array", "newInstance", plus.android.importClass("byte"), bytesArray.length);
					for(let i=0; i<bytesArray.length; i++) {
						plus.android.invoke("java.lang.reflect.Array", "setByte", JavaArray, i, bytesArray[i]);
					}
					outStream.write(JavaArray);
				} else {
					let dataString = text + '\r\n\r\n'; 
					let bytes = plus.android.invoke(dataString, "getBytes", "gbk");
					outStream.write(bytes);
				}

				outStream.flush();
				this.addMessage('send', text, isAuto);
				if (!isAuto) this.inputText = ''; 
			} catch (e) {
				console.error(e);
				this.handleInternalDisconnect();
			}
		},

		sendByBLE(text, isAuto) {
			if (!this.serviceId || !this.characteristicId) {
				this.initBLEService();
				return;
			}
			
			let buffer;
			if (this.isHex) {
				try {
					buffer = this.hexToArrayBuffer(text);
				} catch (e) {
					return uni.showToast({ title: '十六进制格式错误', icon: 'none' });
				}
			} else {
				let dataString = text + '\r\n\r\n';
				buffer = new ArrayBuffer(dataString.length);
				const dataView = new DataView(buffer);
				for (let i = 0; i < dataString.length; i++) {
					dataView.setUint8(i, dataString.charCodeAt(i));
				}
			}

			uni.writeBLECharacteristicValue({
				deviceId: this.deviceId,
				serviceId: this.serviceId,
				characteristicId: this.characteristicId,
				value: buffer,
				writeType: 'writeNoResponse',
				success: () => {
					this.addMessage('send', text, isAuto);
					if (!isAuto) this.inputText = '';
				},
				fail: (err) => { 
					if (err.errCode === 10006) this.handleInternalDisconnect();
					else uni.showToast({ title: '发送失败:' + err.errCode, icon: 'none' });
				}
			});
		},

		toggleAutoRepeat() {
			if (!this.inputText && !this.autoRepeat) return;
			this.autoRepeat = !this.autoRepeat;
		},

		startRepeat() {
			if (this.repeatTimer) clearInterval(this.repeatTimer);
			this.repeatTimer = setInterval(() => { this.sendData(true); }, this.interval);
		},

		stopRepeat() {
			if (this.repeatTimer) { clearInterval(this.repeatTimer); this.repeatTimer = null; }
		},

		initBLEService() {
			uni.getBLEDeviceServices({
				deviceId: this.deviceId,
				success: (res) => {
					const mainService = res.services.find(s => s.uuid.includes('FFE0') || s.uuid.includes('FFF0')) || res.services[0];
					this.serviceId = mainService.uuid;
					uni.getBLEDeviceCharacteristics({
						deviceId: this.deviceId,
						serviceId: this.serviceId,
						success: (cRes) => {
							const writeChar = cRes.characteristics.find(c => c.properties.write || c.properties.writeWithoutResponse);
							if (writeChar) {
								this.characteristicId = writeChar.uuid;
							}
							const notifyChar = cRes.characteristics.find(c => c.properties.notify || c.properties.indicate);
							if (notifyChar) {
								uni.notifyBLECharacteristicValueChange({
									deviceId: this.deviceId,
									serviceId: this.serviceId,
									characteristicId: notifyChar.uuid,
									state: true
								});
							}
						}
					});
				}
			});
		},

		listenBluetoothData() {
			if (this.protocol === 'BLE') {
				uni.onBLECharacteristicValueChange((res) => {
					let str = String.fromCharCode.apply(null, Array.from(new Uint8Array(res.value)));
					// 传递 raw buffer 以支持 HEX 解析
					this.processReceivedData(str, res.value); 
				});
			} else if (this.protocol === 'SPP') {
				const app = getApp();
				if (app.globalData && app.globalData.sppSocket) {
					this.startSPPListen(app.globalData.sppSocket);
				}
			}
		},
		
		startSPPListen(socket) {
			try {
				let inputStream = socket.getInputStream();
				plus.android.importClass(inputStream);
				let InputStreamReader = plus.android.importClass("java.io.InputStreamReader");
				let Scanner = plus.android.importClass("java.util.Scanner");
				let reader = plus.android.newObject("java.io.InputStreamReader", inputStream, "GBK");
				let scanner = plus.android.newObject("java.util.Scanner", reader);
		
				if (this.sppReadTimer) clearInterval(this.sppReadTimer);
		
				this.sppReadTimer = setInterval(() => {
					try {
						if (!this.isConnected) return;
						if (inputStream.available() > 0) {
							if (scanner.hasNext()) {
								let content = scanner.next(); 
								if (content) {
									// SPP Scanner 读出的是字符串，如需底层 HEX 处理建议使用原生 Byte 读取插件
									// 这里先将字符串送入通用处理接口
									let text = content.toString();
									if (this.isHex) text = this.stringToHex(text);
									this.processReceivedData(text);
								}
							}
						}
					} catch (err) {
						this.handleInternalDisconnect();
					}
				}, 150); 
			} catch (e) {
				console.error("SPP监听启动失败", e);
			}
		},

		clearMessages() {
			uni.showModal({
				title: '清空历史',
				content: '确认清空聊天记录吗？',
				success: (res) => {
					if (res.confirm) {
						this.messages = [];
						uni.removeStorageSync('chat_history_' + this.deviceId);
					}
				}
			});
		},
		
		downloadLog() { uni.showToast({ title: '日志已缓存', icon: 'success' }); },
		goBack() { uni.navigateBack(); }
	}
}
</script>

<style scoped>
.chat-container { 
	display: flex; 
	flex-direction: column; 
	height: 100vh; 
	background-color: #f8fafc; 
	overflow: hidden; 
}
.status-bar { height: var(--status-bar-height); background-color: #ffffff; flex-shrink: 0; }
.connection-header { display: flex; justify-content: space-between; align-items: center; background-color: #ffffff; padding: 20rpx 30rpx; border-bottom: 1rpx solid #f1f5f9; flex-shrink: 0; box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.02); z-index: 10;}
.left-section { display: flex; align-items: center; }
.back-icon { font-size: 38rpx; margin-right: 15rpx; color: #64748b; font-weight: bold; }
.conn-status { display: flex; align-items: center; background-color: #f0fdf4; padding: 10rpx 24rpx; border-radius: 40rpx; border: 1px solid #dcfce7; }
.status-dot { width: 14rpx; height: 14rpx; border-radius: 50%; background-color: #94a3b8; margin-right: 12rpx; }
.status-dot.active { background-color: #22c55e; box-shadow: 0 0 10rpx rgba(34, 197, 94, 0.4); }
.conn-text { font-size: 24rpx; color: #166534; font-weight: 700; }
.page-title { font-size: 28rpx; color: #1e293b; font-weight: 800; letter-spacing: 2rpx;}

.chat-window { flex: 1; height: 0; padding: 30rpx 20rpx; box-sizing: border-box; }
.msg-row { display: flex; margin-bottom: 35rpx; }
.msg-row.send { justify-content: flex-end; }
.msg-row.receive { justify-content: flex-start; }
.msg-content { max-width: 78%; padding: 24rpx 32rpx; box-shadow: 0 6rpx 16rpx rgba(0,0,0,0.04); }

/* 深度优化：采用胶囊大圆角气泡设计 */
.send .msg-content { background: linear-gradient(135deg, #3b82f6, #2563eb); color: #ffffff; border-radius: 40rpx 40rpx 8rpx 40rpx; }
.receive .msg-content { background-color: #ffffff; color: #334155; border-radius: 40rpx 40rpx 40rpx 8rpx; border: 1px solid #f1f5f9; }

.msg-text { font-size: 28rpx; word-break: break-all; line-height: 1.6; font-family: monospace; }
.msg-time { font-size: 20rpx; opacity: 0.6; display: block; margin-top: 12rpx; text-align: right; }

.floating-controls { position: fixed; right: 24rpx; top: 40%; display: flex; flex-direction: column; gap: 24rpx; z-index: 99; }
.float-btn { width: 76rpx; height: 76rpx; border-radius: 50%; display: flex; align-items: center; justify-content: center; color: #fff; font-size: 34rpx; font-weight: bold; box-shadow: 0 6rpx 16rpx rgba(0,0,0,0.15); transition: transform 0.2s; }
.float-btn:active { transform: scale(0.9); }
.red { background: linear-gradient(135deg, #ef4444, #dc2626); }
.green { background: linear-gradient(135deg, #10b981, #059669); }

.input-panel { background: #ffffff; padding: 24rpx 30rpx calc(24rpx + env(safe-area-inset-bottom)); border-top: 1px solid #f1f5f9; box-shadow: 0 -4rpx 20rpx rgba(0,0,0,0.02); flex-shrink: 0; z-index: 10; }
.input-row { display: flex; align-items: center; gap: 20rpx; margin-bottom: 24rpx; }

/* 优化：胶囊形态的输入框 */
.capsule-input { flex: 1; background: #f8fafc; height: 88rpx; border-radius: 44rpx; padding: 0 40rpx; font-size: 28rpx; border: 2rpx solid #e2e8f0; transition: all 0.3s ease; }
.capsule-input:focus { border-color: #3b82f6; background: #ffffff; box-shadow: 0 0 0 6rpx rgba(59,130,246,0.1); }
.input-sending { background-color: #fef2f2 !important; color: #ef4444; border-color: #fca5a5 !important; }

.send-btn-circle { width: 88rpx; height: 88rpx; background: linear-gradient(135deg, #3b82f6, #2563eb); border-radius: 50%; color: #fff; display: flex; align-items: center; justify-content: center; font-size: 40rpx; font-weight: bold; box-shadow: 0 6rpx 16rpx rgba(59,130,246,0.3); transition: all 0.2s; }
.send-btn-circle:active { transform: scale(0.92); }
.stop-btn { background: linear-gradient(135deg, #ef4444, #dc2626); box-shadow: 0 6rpx 16rpx rgba(239,68,68,0.3); font-size: 36rpx; }

.config-row { display: flex; align-items: center; justify-content: space-between; padding: 0 10rpx;}
.checkbox-item { display: flex; align-items: center; gap: 12rpx; font-size: 24rpx; color: #64748b; font-weight: 600; padding: 10rpx; }
.custom-radio { width: 32rpx; height: 32rpx; border-radius: 50%; border: 4rpx solid #cbd5e1; transition: all 0.2s; position: relative; }
.radio-active { border-color: #3b82f6; }
.radio-active::after { content: ''; position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 16rpx; height: 16rpx; background: #3b82f6; border-radius: 50%; }
.red-radio.radio-active { border-color: #ef4444; }
.red-radio.radio-active::after { background: #ef4444; }

.text-blue { color: #3b82f6; }
.text-danger { color: #ef4444; }

.repeat-interval { display: flex; align-items: center; background: #f8fafc; padding: 8rpx 24rpx; border-radius: 30rpx; border: 1px solid #e2e8f0; }
.interval-input { width: 80rpx; text-align: center; font-size: 26rpx; color: #3b82f6; font-weight: bold; font-family: monospace; }
.unit-text { font-size: 22rpx; color: #94a3b8; font-weight: bold; }
</style>