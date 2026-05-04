<template>
	<scroll-view :scroll-y="!showConfig" class="container">
		<view class="status-bar"></view>
		
		<view class="track-switcher">
			<view class="track-btn active">🏁 赛道一 (基础巡线)</view>
			<view class="track-btn" @tap="goToTrack2">🚗 赛道二 (算术巡航)</view>
		</view>

		<view class="connection-header" @tap="showDeviceList = !showDeviceList">
			<view class="conn-status">
				<view class="status-dot" :class="{ active: isConnected }"></view>
				<view style="display: flex; align-items: center;">
					<text class="conn-text">{{ isConnected ? ('已连接：' + deviceName) : '未连接设备' }}</text>
					<text class="toggle-hint">{{ showDeviceList ? '收起蓝牙 ▲' : '展开蓝牙 ▼' }}</text>
				</view>
			</view>
			<button class="btn-config-entry" @tap.stop="showConfig = true">⚙ 总体配置</button>
		</view>

		<view class="scan-panel" v-show="showDeviceList">
			<view class="scan-controls">
				<button class="btn-scan" :loading="isScanning" @tap.stop="toggleScan">
					{{ isScanning ? '停止扫描' : '开始扫描' }}
				</button>
				<picker @change="filterChange" :range="protocolOptions" :value="filterIndex">
					<view class="filter-tag">{{ protocolOptions[filterIndex] }} ▼</view>
				</picker>
				<button v-if="isConnected" class="btn-disconnect-sm" @tap.stop="disconnectDevice">断开</button>
			</view>

			<scroll-view scroll-y class="device-list-container" v-if="deviceList.length > 0 || isScanning">
				<view v-if="deviceList.length === 0 && !isScanning" class="empty-box">
					<text class="empty-text">未发现设备，请确保定位与蓝牙已开启</text>
				</view>

				<view class="device-item" v-for="(item, index) in filteredDeviceList" :key="item.deviceId">
					<view class="device-info">
						<text class="device-name">{{ item.name || 'Unknown Device' }}</text>
						<text class="device-mac">{{ item.deviceId }}</text>
						<view class="tags">
							<text class="tag-protocol" :class="item.protocol">{{ item.protocol }}</text>
							<text class="tag-rssi" :class="getRssiClass(item.RSSI)">📶 {{ item.RSSI }}{{ item.RSSI !== 'Unknown' ? 'dBm' : '' }}</text>
							<text v-if="deviceId === item.deviceId" class="tag-connected">已连接</text>
						</view>
					</view>
					<button class="btn-connect" :loading="connectingId === item.deviceId"
						:disabled="deviceId === item.deviceId"
						@tap.stop="connectTo(item)">
						{{ deviceId === item.deviceId ? '已连' : '连接' }}
					</button>
				</view>
			</scroll-view>
		</view>
				
		<view class="card">
			<view class="card-header">
				<text class="header-icon">⏻</text>
				<text class="card-title">赛道一控制面板</text>
			</view>
			
			<button class="btn-main-action" @tap="addCommand('start', '启动运行')">
				<text class="play-icon">▶</text> 启动运行 <text class="code-hint">({{ config.start }}@)</text>
			</button>
			
			<view class="chat-preview-box" @tap="goToChat">
			    <view class="chat-content">
			        <text class="chat-icon">💬</text>
			        <view class="chat-text-area">
			            <text class="chat-label">设备对话收发</text>
			            <text class="chat-hint">点击进入实时对话界面</text>
			        </view>
			    </view>
			    <text class="arrow-right">></text>
			</view>

			<view class="section red-bg" v-if="customCommands.routes && customCommands.routes.length > 0">
				<view class="section-title" @tap="toggleCmdDrawer('routes')" style="justify-content: space-between; cursor: pointer;">
					<view style="display:flex; align-items:center;"><text class="title-icon">🌟</text> 我的预设路线</view>
					<text class="toggle-hint">{{ cmdDrawer.routes ? '收起 ▲' : '展开 ▼' }}</text>
				</view>
				<view class="button-grid-2" v-show="cmdDrawer.routes">
					<button class="grid-btn custom-btn" v-for="(cmd, idx) in customCommands.routes" :key="'croute'+idx" @tap="addCommand(cmd.code, cmd.name, true)">
						{{ cmd.name }}
					</button>
				</view>
			</view>

			<view class="section blue-bg">
				<view class="section-title" @tap="toggleCmdDrawer('lines')" style="justify-content: space-between; cursor: pointer;">
					<view style="display:flex; align-items:center;"><text class="title-arrow">↑</text> 直走到路口</view>
					<text class="toggle-hint">{{ cmdDrawer.lines ? '收起 ▲' : '展开 ▼' }}</text>
				</view>
				<view class="button-grid" v-show="cmdDrawer.lines">
					<button v-for="i in 5" :key="'go'+i" class="grid-btn" @tap="addCommand('go' + i, '直走' + i)">
						↑ 直走{{i}} ({{ config['go'+i] }}@)
					</button>
					<button class="grid-btn custom-btn" v-for="(cmd, idx) in customCommands.lines" :key="'clines'+idx" @tap="addCommand(cmd.code, cmd.name, true)">
						{{ cmd.name }}
					</button>
				</view>
			</view>

			<view class="section cyan-bg">
				<view class="section-title" @tap="toggleCmdDrawer('time')" style="justify-content: space-between; cursor: pointer;">
					<view style="display:flex; align-items:center;"><text class="title-icon">🕒</text> 直走时间</view>
					<text class="toggle-hint">{{ cmdDrawer.time ? '收起 ▲' : '展开 ▼' }}</text>
				</view>
				<view class="button-grid-3" v-show="cmdDrawer.time">
					<button v-for="i in 3" :key="'time'+i" class="grid-btn" @tap="addCommand('time'+i, '直走'+i+'秒')">
						{{i}}秒 ({{ config['time'+i] }}@)
					</button>
					<button class="grid-btn custom-btn" v-for="(cmd, idx) in customCommands.time" :key="'ctime'+idx" @tap="addCommand(cmd.code, cmd.name, true)">
						{{ cmd.name }}
					</button>
				</view>
			</view>

			<view class="section purple-bg">
				<view class="section-title" @tap="toggleCmdDrawer('base')" style="justify-content: space-between; cursor: pointer;">
					<view style="display:flex; align-items:center;">🕹️ 操作指令</view>
					<text class="toggle-hint">{{ cmdDrawer.base ? '收起 ▲' : '展开 ▼' }}</text>
				</view>
				<view class="button-grid-2" v-show="cmdDrawer.base">
					<button class="grid-btn" @tap="addCommand('left', '左拐弯')">← 左拐弯 ({{ config.left }}@)</button>
					<button class="grid-btn" @tap="addCommand('right', '右拐弯')">→ 右拐弯 ({{ config.right }}@)</button>
					<button class="grid-btn" @tap="addCommand('stop', '停止')">🔲 停止 ({{ config.stop }}@)</button>
					<button class="grid-btn" @tap="addCommand('light', '亮灯')">💡 亮灯 ({{ config.light }}@)</button>
					<button class="grid-btn custom-btn" v-for="(cmd, idx) in customCommands.base" :key="'cbase'+idx" @tap="addCommand(cmd.code, cmd.name, true)">
						{{ cmd.name }}
					</button>
				</view>
			</view>

			<view class="section orange-bg">
				<view class="section-title" @tap="toggleCmdDrawer('advanced')" style="justify-content: space-between; cursor: pointer;">
					<view style="display:flex; align-items:center;"><text class="title-icon">📍</text> 路径规划动作集</view>
					<text class="toggle-hint">{{ cmdDrawer.advanced ? '收起 ▲' : '展开 ▼' }}</text>
				</view>
				<view class="button-grid-3" v-show="cmdDrawer.advanced">
					<button class="grid-btn" @tap="addCommand('crossLeft', '左转(横向移动)')">横向左转 ({{ config.crossLeft }}@)</button>
					<button class="grid-btn" @tap="addCommand('crossRight', '右转(横向移动)')">横向右转 ({{ config.crossRight }}@)</button>
					<button class="grid-btn" @tap="addCommand('findCol', '寻线至正下方')">寻列下方 ({{ config.findCol }}@)</button>
					
					<button class="grid-btn" @tap="addCommand('alignLeft', '左转回正(面向上方)')">左转回正 ({{ config.alignLeft }}@)</button>
					<button class="grid-btn" @tap="addCommand('alignRight', '右转回正(面向上方)')">右转回正 ({{ config.alignRight }}@)</button>
					<button class="grid-btn" @tap="addCommand('enterBlock', '直行进入方块中心')">进块中心 ({{ config.enterBlock }}@)</button>
					
					<button class="grid-btn" @tap="addCommand('pauseBlock', '中心停留1秒')">停留1秒 ({{ config.pauseBlock }}@)</button>
					<button class="grid-btn" @tap="addCommand('exitBlock', '直行穿出到上方路口')">直穿出块 ({{ config.exitBlock }}@)</button>
					<button class="grid-btn" @tap="addCommand('goFinish', '直行进入终点区')">进终点区 ({{ config.goFinish }}@)</button>
					<button class="grid-btn custom-btn" v-for="(cmd, idx) in customCommands.advanced" :key="'cadv'+idx" @tap="addCommand(cmd.code, cmd.name, true)">
						{{ cmd.name }}
					</button>
				</view>
			</view>

			<button class="btn-delete" @tap="deleteLast">
				<text class="del-icon">⌫</text> 删除上一条指令
			</button>
		</view>

		<view class="card">
			<view class="card-header">
				<text class="header-icon">🗺️</text>
				<text class="card-title">智能路径规划</text>
			</view>
			
			<view class="path-planning-box">
				<view class="mode-tabs">
					<view class="mode-tab" :class="{active: planMode === 'auto'}" @tap="switchMode('auto')">🤖 自动算路</view>
					<view class="mode-tab" :class="{active: planMode === 'manual'}" @tap="switchMode('manual')">👆 手动点选</view>
				</view>

				<view class="input-group">
					<text class="input-label">选择启动点 (1-6)</text>
					<picker @change="onStartPointChange" :range="[1,2,3,4,5,6]" :value="startPointIndex">
						<view class="picker-display">{{ startPointIndex + 1 }} 号位置 ▼</view>
					</picker>
				</view>
				
				<block v-if="planMode === 'auto'">
					<view class="input-group">
						<text class="input-label">目标相加总和</text>
						<input type="number" v-model.number="targetSum" class="sum-input" placeholder="输入要计算的值 (如: 16)" />
					</view>
					<button class="btn-generate-path" @tap="calculateAndGenerate">生成最短路径指令</button>
				</block>

				<block v-if="planMode === 'manual'">
					<view class="input-group">
						<text class="input-label">已选数值总和</text>
						<text class="picker-display">{{ manualCurrentSum }}</text>
					</view>
					<view class="manual-hint">💡 请直接点击下方地图，为每行选择1个方块</view>
					<button class="btn-generate-path manual-btn" @tap="generateManualPathCmd">生成手绘路径指令</button>
				</block>

				<view v-if="bestPath.length > 0" class="path-result">
					<view class="result-row">
						<text class="result-tag">最优途径列</text>
						<text class="result-val">{{ bestPath.map(c => c + 1).join(' ➔ ') }}</text>
					</view>
					<view class="result-row">
						<text class="result-tag">对应块数值</text>
						<text class="result-val">{{ pathValues.join(' + ') }} = {{ planMode === 'auto' ? targetSum : manualCurrentSum }}</text>
					</view>
				</view>

				<view class="input-group" style="flex-direction: column; align-items: flex-start; margin-top: 20rpx;">
					<text class="input-label" style="margin-bottom: 16rpx; color: #d97706;">
						💡 路径发送格式预览 (可在此修改转弯L/R)
					</text>
					<textarea v-model="pathMacroPreview" placeholder="点击上方“生成指令”后，将在此显示路线编码... 您可以直接编辑它。" style="width: 100%; height: 160rpx; background: #fff; padding: 20rpx; border: 2rpx solid #cbd5e1; border-radius: 12rpx; font-size: 28rpx; color: #1e293b; font-family: monospace; box-sizing: border-box;" :maxlength="-1"></textarea>
					<button class="btn-generate-path" style="width: 100%; margin-top: 16rpx; background: #10b981; color: #fff; border: none;" @tap="applyMacroToQueue">覆盖并应用到发送队列</button>
				</view>
			</view>
		</view>
		
		<view class="card">
			<view class="card-header-between">
				<text class="card-title">指令队列</text>
				<view>
					<text class="text-link text-blue" style="margin-right: 24rpx;" @tap="saveQueueAsMacro">💾 存为预设</text>
					<text class="text-link" @tap="clearQueue">清空</text>
				</view>
			</view>
			<view class="queue-display-area">
				<view v-if="commandQueue.length === 0" class="empty-placeholder">
					<view class="empty-text">暂无指令</view>
					<view class="empty-subtext">请通过控制面板或路径规划添加指令</view>
				</view>
				<view v-else class="queue-list">
					<view class="queue-item" v-for="(item, index) in commandQueue" :key="index">
						<text class="q-name">{{ item.name }}</text>
						<text class="q-code" v-if="item.code">[{{ item.code }}]</text>
					</view>
				</view>
			</view>
			<button class="btn-send-active" @tap="sendBatch">
				<text class="send-icon">🚀</text> 发送指令 ({{ commandQueue.length }})
			</button>
		</view>

		<view class="card">
			<view class="card-header-between">
				<text class="card-title">赛道一运动地图</text>
				<text class="text-link" @tap="clearMap">🗑️ 清空路线</text>
			</view>

		    <view class="map-container">
		        <view class="map-header-row">
		            <view class="row-side-label no-border"></view> 
		            <view class="end-point-bar">
		                <view class="v-line" v-for="i in 4" :key="'t'+i"></view>
		                <view class="bar-text">
		                    <text></text> <text>终</text>
		                    <text></text> <text>点</text>
		                    <text></text> </view>
		            </view>
		        </view>
		        
		        <view class="map-body">
					<view class="path-overlay" v-if="pathLines.length > 0">
						<view class="path-line" v-for="(line, index) in pathLines" :key="'line'+index" 
							  :style="{ left: line.left, top: line.top, width: line.width, height: line.height }">
						</view>
					</view>

		            <view class="grid-row" v-for="row in [5,4,3,2,1]" :key="row">
		                <text class="row-side-label">路径点{{row}}</text>
		                <view class="grid-cell" v-for="col in 5" :key="col" 
							  :class="{'path-active': (planMode === 'auto' && bestPath.length > 0 && bestPath[row - 1] === (col - 1)) || (planMode === 'manual' && manualPath[row - 1] === (col - 1))}"
							  @tap="onCellTap(row, col)">
							
		                    <text class="cell-num">{{col}}</text>
		                    
							<view class="nested-box box-red">
		                        <view class="nested-box box-yellow">
		                            <view class="nested-box box-green"></view>
		                        </view>
		                    </view>
		                </view>
		            </view>
		        </view>
		
		        <view class="map-footer-row">
		            <view class="row-side-label no-border"></view>
		            <view class="start-zone-bar">
		                <view class="v-line" v-for="i in 4" :key="'b'+i"></view>
		                <view class="bar-text">
		                    <text>启</text>
		                    <text></text> <text>动</text>
		                    <text></text> <text>区</text>
		                </view>
		            </view>
		        </view>
				
				<view class="map-axis-row">
				    <view class="row-label-empty no-border"></view> <view class="axis-content">
				        <view class="v-line-ext" v-for="i in 6" :key="'line'+i"></view>
				        <view class="axis-numbers">
				            <text v-for="n in 6" :key="'num'+n">{{n}}</text>
				        </view>
				    </view>
				</view>
		    </view>
		</view>

		<view class="config-mask" v-if="showConfig" @tap.self="showConfig = false" @touchmove.stop.prevent>
			<view class="config-panel" @touchmove.stop>
				<view class="config-title">总体配置 - 设置发送字符</view>
				
				<scroll-view scroll-y="true" class="config-scroll">

					<view class="config-collapse-header" @tap="toggleConfigTab('custom')">
						<text>🛠️ 自定义指令管理</text><text>{{ configTabs.custom ? '▲' : '▼' }}</text>
					</view>
					<view class="config-collapse-body" v-show="configTabs.custom">
						<view class="custom-cmd-form">
							<picker @change="onCustomModuleChange" :range="moduleOptions" range-key="name" :value="customModuleIndex">
								<view class="custom-picker">添加至：<text class="text-blue">{{ moduleOptions[customModuleIndex].name }}</text> ▼</view>
							</picker>
							<view class="custom-input-row">
								<input class="c-input" v-model="newCustomName" placeholder="按钮名称(例:倒车)" />
								<input class="c-input" v-model="newCustomCode" placeholder="代码(例:BACK)" />
							</view>
							<button class="btn-add-custom" @tap="addCustomCommandToStorage">➕ 添加至面板</button>
						</view>

						<view class="custom-list">
							<view class="custom-list-item" v-for="(modName, modKey) in moduleNamesMap" :key="modKey" v-if="customCommands[modKey] && customCommands[modKey].length > 0">
								<view class="custom-mod-title">【{{ modName }}】：</view>
								<view class="custom-cmd-row" v-for="(cmd, cIdx) in customCommands[modKey]" :key="cIdx">
									<text class="cc-name">{{ cmd.name }}</text>
									<text class="cc-del" @tap="removeCustomCommand(modKey, cIdx)">❌ 删除</text>
								</view>
							</view>
						</view>
					</view>
					
					<view class="config-collapse-header" @tap="toggleConfigTab('base')">
						<text>🕹️ 基础运动指令</text><text>{{ configTabs.base ? '▲' : '▼' }}</text>
					</view>
					<view class="config-collapse-body" v-show="configTabs.base">
						<view class="config-item"><text>启动运行</text><input v-model="config.start" /></view>
						<view class="config-item"><text>左拐弯</text><input v-model="config.left" /></view>
						<view class="config-item"><text>右拐弯</text><input v-model="config.right" /></view>
						<view class="config-item"><text>停止</text><input v-model="config.stop" /></view>
						<view class="config-item"><text>亮灯</text><input v-model="config.light" /></view>
					</view>

					<view class="config-collapse-header" @tap="toggleConfigTab('lines')">
						<text>📏 直走跨线指令</text><text>{{ configTabs.lines ? '▲' : '▼' }}</text>
					</view>
					<view class="config-collapse-body" v-show="configTabs.lines">
						<view class="config-item" v-for="i in 5" :key="'cgo'+i">
							<text>直走{{i}}</text><input v-model="config['go'+i]" />
						</view>
					</view>

					<view class="config-collapse-header" @tap="toggleConfigTab('time')">
						<text>🕒 直走延时指令</text><text>{{ configTabs.time ? '▲' : '▼' }}</text>
					</view>
					<view class="config-collapse-body" v-show="configTabs.time">
						<view class="config-item" v-for="i in 3" :key="'ctm'+i">
							<text>直走{{i}}秒</text><input v-model="config['time'+i]" />
						</view>
					</view>

					<view class="config-collapse-header" @tap="toggleConfigTab('advanced')">
						<text>📍 高级专用动作代码</text><text>{{ configTabs.advanced ? '▲' : '▼' }}</text>
					</view>
					<view class="config-collapse-body" v-show="configTabs.advanced">
						<view class="config-item"><text>横向左转</text><input v-model="config.crossLeft" /></view>
						<view class="config-item"><text>横向右转</text><input v-model="config.crossRight" /></view>
						<view class="config-item"><text>寻线至列正下方</text><input v-model="config.findCol" /></view>
						<view class="config-item"><text>左转回正(面向上)</text><input v-model="config.alignLeft" /></view>
						<view class="config-item"><text>右转回正(面向上)</text><input v-model="config.alignRight" /></view>
						<view class="config-item"><text>进方块中心</text><input v-model="config.enterBlock" /></view>
						<view class="config-item"><text>中心停留</text><input v-model="config.pauseBlock" /></view>
						<view class="config-item"><text>穿出到上方路口</text><input v-model="config.exitBlock" /></view>
						<view class="config-item"><text>直行进入终点区</text><input v-model="config.goFinish" /></view>
					</view>

					<view style="height: 40rpx;"></view>
				</scroll-view>
				
				<view class="config-footer">
					<button class="btn-exit-config" @tap="showConfig = false">退出</button>
					<button class="btn-save-config" @tap="saveConfig">保存并应用</button>
				</view>
			</view>
		</view>
		
		<view class="safe-bottom"></view>
	</scroll-view>
</template>

<script>
export default {
    data() {
        return {
			showDeviceList: false, 
			isScanning: false,
			connectingId: '', 
			filterIndex: 0,
			protocolOptions: ['全部协议', '仅BLE', '仅SPP'],
			deviceList: [],
			deviceModel: '',
			isAndroid: uni.getSystemInfoSync().platform === 'android',
			sppCheckTimer: null, 
			bleHeartbeatTimer: null,
			sppReceiver: null, 
			
			bleStateChangeCallback: null,
			
            deviceName: '未连接设备',
            deviceId: '',         
            serviceId: '',        
            characteristicId: '', 
            protocol: '',         
            isConnected: false,   
            showConfig: false,
            commandQueue: [], 
            
            config: {
                start: 'S', 
                stop: 'P',  
                left: '8',  
                right: 'B', 
                light: 'D', 
                go1: '1', go2: '2', go3: '3', go4: '1', go5: '2',
                time1: '1', time2: '2', time3: '3',
				crossLeft: '7',      
				crossRight: 'A',     
				findCol: 'S',        
				alignLeft: '8',      
				alignRight: 'B',     
				enterBlock: '1',     
				pauseBlock: 'P',     
				exitBlock: '1',      
				goFinish: '3'        
            },
			
			configTabs: {
				custom: false, base: true, lines: false, time: false, advanced: false
			},

			cmdDrawer: {
				routes: false,
				lines: false,
				time: false,
				base: false,
				advanced: false
			},
			
			customModuleIndex: 0,
			moduleOptions: [
				{ id: 'base', name: '操作指令' },
				{ id: 'lines', name: '直走到路口' },
				{ id: 'time', name: '直走时间' },
				{ id: 'advanced', name: '路径规划动作集' },
				{ id: 'routes', name: '我的预设路线' } 
			],
			moduleNamesMap: {
				'base': '操作指令', 'lines': '直走到路口', 'time': '直走时间', 'advanced': '路径规划动作集', 'routes': '我的预设路线'
			},
			newCustomName: '',
			newCustomCode: '',
			customCommands: {
				base: [], lines: [], time: [], advanced: [], routes: [] 
			},
			
			planMode: 'auto', 
			manualPath: [-1, -1, -1, -1, -1], 
			manualCurrentSum: 0, 
			
			targetSum: null,
			startPointIndex: 4, 
			bestPath: [],   
			pathValues: [],
			pathLines: [],
			pathMacroPreview: '' 
        }
    },
	computed: {
		filteredDeviceList() {
			if (this.filterIndex === 1) return this.deviceList.filter(d => d.protocol === 'BLE');
			if (this.filterIndex === 2) return this.deviceList.filter(d => d.protocol === 'SPP');
			return this.deviceList;
		}
	},
    onLoad(options) {
        if (options.deviceId) this.deviceId = options.deviceId;
        if (options.protocol) this.protocol = options.protocol;
        if (options.deviceName) this.deviceName = decodeURIComponent(options.deviceName);

		const app = getApp();
		if (!app.globalData) app.globalData = {};
		if (app.globalData.btState && app.globalData.btState.isConnected) {
			this.isConnected = app.globalData.btState.isConnected;
			this.deviceId = app.globalData.btState.deviceId;
			this.deviceName = app.globalData.btState.deviceName;
			this.protocol = app.globalData.btState.protocol;
			this.serviceId = app.globalData.btState.serviceId;
			this.characteristicId = app.globalData.btState.characteristicId;
		}

		this.initBluetooth();
		this.registerSppReceiver();

        if (this.protocol === 'SPP') {
            if (app.globalData && app.globalData.sppSocket) {
                this.isConnected = true;
				this.startSPPCheck();
            }
        } else if (this.protocol === 'BLE' && this.isConnected) {
			this.startBLECheck();
		}

		uni.$off('bluetoothDisconnected');
        uni.$on('bluetoothDisconnected', () => {
            this.handleInternalDisconnect('蓝牙通信链路已断开');
        });

		this.bleStateChangeCallback = (res) => {
            if (!res.connected && this.deviceId && res.deviceId.toLowerCase() === this.deviceId.toLowerCase()) {
                this.handleInternalDisconnect('BLE 设备已断开');
            }
        };
        uni.onBLEConnectionStateChange(this.bleStateChangeCallback);

        const localConfig = uni.getStorageSync('bt_char_config_track1');
        if (localConfig && localConfig.start !== undefined) {
			this.config = Object.assign(this.config, localConfig);
		} else {
			uni.setStorageSync('bt_char_config_track1', this.config);
		}
		
		const localCustom = uni.getStorageSync('bt_custom_commands_track1');
		if (localCustom) this.customCommands = Object.assign(this.customCommands, localCustom);
		
		const savedQueueState = uni.getStorageSync('bt_queue_state_track1');
		if (savedQueueState) this.commandQueue = savedQueueState;
    },
    onUnload() {
        uni.$off('bluetoothDisconnected');
		if (this.bleStateChangeCallback) {
			uni.offBLEConnectionStateChange(this.bleStateChangeCallback);
		}
		this.clearAllTimers();
		this.unregisterSppReceiver();
		this.stopScan();
    },

    methods: {
		goToTrack2() {
			const app = getApp();
			if (!app.globalData) app.globalData = {};
			app.globalData.btState = {
				isConnected: this.isConnected,
				deviceId: this.deviceId,
				deviceName: this.deviceName,
				protocol: this.protocol,
				serviceId: this.serviceId,
				characteristicId: this.characteristicId
			};
			uni.redirectTo({
				url: '/pages/saidao2/saidao2',
				fail: () => { uni.showToast({ title: '跳转失败', icon: 'none' }); }
			});
		},

		initBluetooth() {
			uni.openBluetoothAdapter({
				success: () => console.log('蓝牙初始化成功'),
				fail: () => {
					uni.showModal({ 
						title: '蓝牙异常', 
						content: '请确认系统蓝牙已打开，且应用已获取位置信息权限。是否前往设置？', 
						success: (res) => {
							if (res.confirm && this.isAndroid) {
								try {
									let main = plus.android.runtimeMainActivity();
									let Intent = plus.android.importClass('android.content.Intent');
									let Settings = plus.android.importClass('android.provider.Settings');
									let intent = new Intent(Settings.ACTION_BLUETOOTH_SETTINGS);
									main.startActivity(intent);
								} catch (e) { console.log("跳转设置失败"); }
							}
						}
					});
				}
			});
		},
		
		toggleScan() {
			this.showDeviceList = true;
			if (this.isScanning) {
				this.stopScan();
			} else {
				uni.getBluetoothAdapterState({
					success: (res) => {
						if (res.available) this.startScan();
						else { uni.showToast({ title: '蓝牙不可用', icon: 'none' }); this.initBluetooth(); }
					},
					fail: () => { this.initBluetooth(); }
				});
			}
		},
		
		startScan() {
			this.deviceList = [];
			this.isScanning = true;
			
			uni.startBluetoothDevicesDiscovery({
				allowDuplicatesKey: false,
				success: () => {
					uni.onBluetoothDeviceFound((res) => {
						res.devices.forEach(device => {
							if (!device.name && !device.localName) return;
							const existIndex = this.deviceList.findIndex(d => d.deviceId === device.deviceId);
							if (existIndex > -1) {
								this.$set(this.deviceList[existIndex], 'RSSI', device.RSSI);
							} else {
								device.protocol = 'BLE';
								this.deviceList.unshift(device);
							}
						});
					});
				}
			});

			if (this.isAndroid) { setTimeout(() => { this.getAndroidPairedDevices(); }, 300); }
			setTimeout(() => this.stopScan(), 15000);
		},

		getAndroidPairedDevices() {
			try {
				let BluetoothAdapter = plus.android.importClass("android.bluetooth.BluetoothAdapter");
				let BAdapter = BluetoothAdapter.getDefaultAdapter();
				if (!BAdapter || !BAdapter.isEnabled()) return;

				let pairedDevices = BAdapter.getBondedDevices();
				if (pairedDevices) {
					plus.android.importClass("android.bluetooth.BluetoothDevice");
					let list = plus.android.invoke(pairedDevices, "toArray");
					if (list && list.length > 0) {
						for (let i = 0; i < list.length; i++) {
							let device = list[i];
							plus.android.importClass(device); 
							let mac = device.getAddress();
							let name = device.getName() || '经典蓝牙设备';
							if (!this.deviceList.find(d => d.deviceId === mac)) {
								this.deviceList.push({ name: name + ' (已配对)', deviceId: mac, protocol: 'SPP', RSSI: 'Unknown', isPaired: true });
							}
						}
					}
				}
			} catch (e) {}
		},

		stopScan() {
			if(this.isScanning) { this.isScanning = false; uni.stopBluetoothDevicesDiscovery(); }
		},

		connectTo(device) {
			if (this.isConnected) return uni.showToast({ title: '请先断开当前连接', icon: 'none' });
			this.connectingId = device.deviceId;
			this.stopScan(); 
			
			if (device.protocol === 'SPP') {
				this.connectSPP(device);
			} else {
				uni.createBLEConnection({
					deviceId: device.deviceId,
					timeout: 10000,
					success: () => {
						this.deviceId = device.deviceId;
						this.deviceName = device.name || '未知设备';
						this.protocol = 'BLE';
						this.isConnected = true;
						this.showDeviceList = false;
						
						this.initBLEService();
						this.startBLECheck();
						uni.showToast({ title: '连接成功' });
					},
					fail: (err) => { uni.showModal({ title: '连接失败', content: '错误码: ' + err.errCode, showCancel: false }); },
					complete: () => { this.connectingId = ''; }
				});
			}
		},

		connectSPP(device) {
			setTimeout(() => {
				try {
					let BluetoothAdapter = plus.android.importClass("android.bluetooth.BluetoothAdapter");
					let BAdapter = BluetoothAdapter.getDefaultAdapter();
					let remoteDevice = BAdapter.getRemoteDevice(device.deviceId);
					let UUID = plus.android.importClass("java.util.UUID");
					let sppUuid = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB");
					
					let socket = remoteDevice.createRfcommSocketToServiceRecord(sppUuid);
					plus.android.importClass(socket);
					socket.connect();
					
					getApp().globalData.sppSocket = socket;
					this.deviceId = device.deviceId;
					this.deviceName = device.name || '经典蓝牙串口';
					this.protocol = 'SPP';
					this.isConnected = true;
					this.showDeviceList = false;
					
					this.startSPPCheck();
					uni.showToast({ title: 'SPP连接成功' });
				} catch (e) {
					uni.showModal({ title: '连接失败', content: '请确保设备已配对并开启', showCancel: false });
				}
				this.connectingId = '';
			}, 200);
		},

		startSPPCheck() {
			if (this.sppCheckTimer) clearInterval(this.sppCheckTimer);
			this.sppCheckTimer = setInterval(() => {
				if (getApp().globalData.sppSocket && this.isConnected) {
					try {
						let alive = plus.android.invoke(getApp().globalData.sppSocket, "isConnected");
						if (!alive) throw "Socket Disconnected";
					} catch (e) { this.handleInternalDisconnect('SPP 蓝牙链路已失效'); }
				}
			}, 3000); 
		},

		registerSppReceiver() {
		    if (!this.isAndroid) return;
		    try {
		        let main = plus.android.runtimeMainActivity();
		        let IntentFilter = plus.android.importClass('android.content.IntentFilter');
		        let BluetoothDevice = plus.android.importClass('android.bluetooth.BluetoothDevice');
		        
		        this.sppReceiver = plus.android.implements('io.dcloud.feature.internal.reflect.BroadcastReceiver', {
		            onReceive: (context, intent) => {
		                plus.android.importClass(intent);
		                let action = intent.getAction();
		                if (action == BluetoothDevice.ACTION_ACL_DISCONNECTED) {
		                    let device = intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE);
		                    if (device) {
		                        plus.android.importClass(device);
		                        let mac = plus.android.invoke(device, "getAddress");
		                        if (this.isConnected && mac === this.deviceId) {
		                            this.handleInternalDisconnect('设备已物理断开');
		                        }
		                    }
		                }
		            }
		        });
		        
		        let filter = new IntentFilter();
		        filter.addAction(BluetoothDevice.ACTION_ACL_DISCONNECTED);
		        main.registerReceiver(this.sppReceiver, filter);
		    } catch (e) {}
		},
		
		unregisterSppReceiver() {
		    if (this.isAndroid && this.sppReceiver) {
		        try {
		            let main = plus.android.runtimeMainActivity();
		            main.unregisterReceiver(this.sppReceiver);
		            this.sppReceiver = null;
		        } catch (e) {}
		    }
		},

		startBLECheck() {
			if (this.bleHeartbeatTimer) clearInterval(this.bleHeartbeatTimer);
			this.bleHeartbeatTimer = setInterval(() => {
				if (this.isConnected && this.protocol === 'BLE') {
					uni.getBLEDeviceRSSI({ deviceId: this.deviceId, fail: () => this.handleInternalDisconnect('BLE 连接已丢失') });
				}
			}, 5000);
		},

        handleInternalDisconnect(msg = '连接已断开') {
            this.isConnected = false;
            this.deviceName = '未连接设备';

			const app = getApp();
			if (app.globalData) app.globalData.btState = null;

			if (this.protocol === 'BLE' && this.deviceId) {
				uni.closeBLEConnection({ deviceId: this.deviceId });
			}

            if (app.globalData && app.globalData.sppSocket) {
				try { app.globalData.sppSocket.close(); } catch (e) {}
				app.globalData.sppSocket = null;
			}
			
			this.deviceId = '';
			this.protocol = '';
			this.clearAllTimers();
            uni.showModal({ title: '提示', content: msg, showCancel: false });
        },
		
		disconnectDevice() {
			if (!this.isConnected) return;
			this.handleInternalDisconnect('已主动断开连接');
		},

		clearAllTimers() {
			if (this.sppCheckTimer) clearInterval(this.sppCheckTimer);
			if (this.bleHeartbeatTimer) clearInterval(this.bleHeartbeatTimer);
		},

		getRssiClass(rssi) {
			if(rssi === 'Unknown') return 'rssi-unknown';
			let val = parseInt(rssi);
			if(isNaN(val)) return 'rssi-unknown';
			if(val >= -60) return 'rssi-good';   
			if(val >= -80) return 'rssi-normal'; 
			return 'rssi-bad';                   
		},
		
		filterChange(e) { this.filterIndex = parseInt(e.detail.value); },

		toggleCmdDrawer(drawerName) {
			this.$set(this.cmdDrawer, drawerName, !this.cmdDrawer[drawerName]);
		},

		clearMap() {
			this.bestPath = [];
			this.pathLines = [];
			this.pathValues = [];
			this.manualPath = [-1, -1, -1, -1, -1];
			this.manualCurrentSum = 0;
			this.pathMacroPreview = ''; 
			uni.showToast({ title: '地图已清空', icon: 'none' });
		},

		switchMode(mode) {
			this.planMode = mode;
			this.bestPath = [];
			this.pathLines = [];
			this.pathValues = [];
			this.pathMacroPreview = ''; 
			if (mode === 'manual') {
				this.manualPath = [-1, -1, -1, -1, -1];
				this.manualCurrentSum = 0;
			}
		},
		
		onCellTap(row, col) {
			if (this.planMode !== 'manual') return;
			let rowIndex = row - 1;
			let colIndex = col - 1;
			this.$set(this.manualPath, rowIndex, colIndex);
			
			let sum = 0;
			this.manualPath.forEach(cIndex => { if (cIndex !== -1) sum += (cIndex + 1); });
			this.manualCurrentSum = sum;
		},
		
		generateManualPathCmd() {
			if (this.manualPath.includes(-1)) return uni.showToast({ title: '请在地图上将5行全部选满！', icon: 'none' });
			
			this.bestPath = [...this.manualPath];
			this.pathValues = this.bestPath.map(col => col + 1);
			this.targetSum = this.manualCurrentSum; 
			
			this.generateQueueFromPath(this.bestPath, this.startPointIndex);
			this.generatePathLines(this.bestPath, this.startPointIndex);
			
			uni.showToast({ title: '手绘路线已生成预览！', icon: 'success' });
		},

		generateQueueFromPath(path, startLine) {
			let currentX = startLine + 1; 
			let macroSteps = [];

			path.forEach((targetCol, rowIndex) => {
				let targetX = targetCol + 1; 
				let diff = targetX - currentX;
				let codeStr = '';
				
				if (diff > 0) {
					codeStr = `${targetX}(1,R,${Math.abs(diff)},W)`;
				} else if (diff < 0) {
					codeStr = `${targetX}(1,L,${Math.abs(diff)},W)`;
				} else {
					codeStr = `${targetX}(1,W)`;
				}
				
				macroSteps.push(codeStr);
				currentX = targetX; 
			});

			macroSteps.push(`6(1,W)`);
			
			this.pathMacroPreview = macroSteps.join(',') + '@';
		},

		applyMacroToQueue() {
			if (!this.pathMacroPreview) return uni.showToast({ title: '没有可用的路径指令', icon: 'none' });
			
			let q = []; 
			q.push({ name: '完整路径宏指令', code: this.pathMacroPreview });
			
			const startSymbol = this.config.start || 'S';
			let startCode = startSymbol.endsWith('@') ? startSymbol : `${startSymbol}@`;
			q.push({ name: '自动触发启动', code: startCode });

			this.commandQueue = q;
			this.saveQueueState();
			uni.showToast({ title: '已覆盖并应用到发送队列', icon: 'success' });
		},

		calculateAndGenerate() {
			if (!this.targetSum) return uni.showToast({ title: '请输入目标总和值', icon: 'none' });

			let validPaths = [];
			const dfs = (rowIndex, currentSum, currentPath) => {
				if (rowIndex === 5) {
					if (currentSum === this.targetSum) validPaths.push([...currentPath]);
					return;
				}
				for (let col = 0; col < 5; col++) {
					let val = col + 1; 
					if (currentSum + val <= this.targetSum) {
						currentPath.push(col);
						dfs(rowIndex + 1, currentSum + val, currentPath);
						currentPath.pop();
					}
				}
			};
			dfs(0, 0, []);

			if (validPaths.length === 0) {
				uni.showToast({ title: '未找到满足数值的路径', icon: 'none' });
				this.bestPath = []; this.pathValues = []; this.pathLines = [];
				this.pathMacroPreview = '';
				return;
			}

			let minDistance = Infinity;
			let best = [];
			let startLine = this.startPointIndex; 
			
			validPaths.forEach(path => {
				let dist = 0, currentLine = startLine; 
				for (let i = 0; i < path.length; i++) {
					dist += Math.abs(currentLine - path[i]);
					currentLine = path[i];
				}
				if (dist < minDistance) { minDistance = dist; best = path; }
			});

			this.bestPath = best;
			this.pathValues = best.map(col => col + 1);
			
			this.generateQueueFromPath(best, startLine);
			this.generatePathLines(best, startLine);
			
			uni.showToast({ title: '已生成最优路线及预览指令！', icon: 'none' });
		},

		saveQueueState() { uni.setStorageSync('bt_queue_state_track1', this.commandQueue); },
		clearQueue() { this.commandQueue = []; this.saveQueueState(); },

		saveQueueAsMacro() {
			if (this.commandQueue.length === 0) return uni.showToast({ title: '队列为空', icon: 'none' });
			// 保存时无缝拼接即可，我们在解析时会通过 @ 巧妙拆开
			const codes = this.commandQueue.map(i => i.code).join('');
			const defaultName = this.targetSum ? `目标${this.targetSum}路线` : `路线_${new Date().getSeconds()}`;
			
			uni.showModal({
				title: '保存为预设路线',
				content: defaultName,
				editable: true,
				placeholderText: '给这条路线起个名字',
				success: (res) => {
					if (res.confirm) {
						const finalName = res.content || defaultName;
						if (!this.customCommands.routes) this.$set(this.customCommands, 'routes', []);
						this.customCommands.routes.push({ name: finalName, code: codes });
						this.saveCustomCommands();
						uni.showToast({ title: '已保存至[我的预设路线]', icon: 'success' });
					}
				}
			});
		},

		onCustomModuleChange(e) { this.customModuleIndex = parseInt(e.detail.value); },
		addCustomCommandToStorage() {
			if (!this.newCustomName || !this.newCustomCode) return uni.showToast({ title: '请填写名称和代码', icon: 'none' });
			const modId = this.moduleOptions[this.customModuleIndex].id;
			this.customCommands[modId].push({ name: this.newCustomName, code: this.newCustomCode });
			this.newCustomName = ''; this.newCustomCode = '';
			this.saveCustomCommands();
			uni.showToast({ title: '添加成功，已渲染至面板', icon: 'success' });
		},
		removeCustomCommand(modKey, idx) {
			this.customCommands[modKey].splice(idx, 1);
			this.saveCustomCommands();
		},
		saveCustomCommands() { uni.setStorageSync('bt_custom_commands_track1', this.customCommands); },

		toggleConfigTab(tabName) { this.configTabs[tabName] = !this.configTabs[tabName]; },

        sendSPPData(text, isSilent = false) {
            const app = getApp();
            if (!app.globalData.sppSocket) return this.handleInternalDisconnect('连接已失效');
            try {
                let outStream = app.globalData.sppSocket.getOutputStream();
                plus.android.importClass(outStream);
                let bytes = plus.android.invoke(text, "getBytes", "gbk");
                outStream.write(bytes);
                outStream.flush();
				
				if (!isSilent) {
					uni.showToast({ title: '已发送' });
					this.syncToChatCache('send', text.replace(/\r\n/g, ''));
				}
            } catch (e) {
                this.handleInternalDisconnect('设备已掉线，发送失败');
            }
        },

        sendBLEData(text, isSilent = false) {
            if (!this.serviceId || !this.characteristicId) {
                if (!isSilent) uni.showToast({ title: '初始化特征值...', icon: 'none' });
                this.initBLEService();
                return;
            }
            const buffer = new ArrayBuffer(text.length);
            const dataView = new DataView(buffer);
            for (let i = 0; i < text.length; i++) dataView.setUint8(i, text.charCodeAt(i));
            
            uni.writeBLECharacteristicValue({
                deviceId: this.deviceId, serviceId: this.serviceId, characteristicId: this.characteristicId, value: buffer,
                success: () => {
					if (!isSilent) {
						uni.showToast({ title: '已发送' });
						this.syncToChatCache('send', text.replace(/\r\n/g, ''));
					}
                },
                fail: (err) => {
                    if (err.errCode === 10006) this.handleInternalDisconnect('BLE 连接已断开');
                    else if (!isSilent) uni.showToast({ title: '发送失败:' + err.errCode, icon: 'none' });
                }
            });
        },

		// 核心优化：彻底分行发送，并且等待400ms，没有开头的多余 K 指令
        async sendBatch() {
            if (this.commandQueue.length === 0) return;
            if (!this.isConnected) return uni.showToast({ title: '蓝牙未连接', icon: 'none' });
			
			for (let i = 0; i < this.commandQueue.length; i++) {
				let cmd = this.commandQueue[i].code;
				
				// 逐行发送，系统底层会自动追加 \r\n，避免连线粘包
				const dataString = cmd + '\r\n'; 
				
				if (this.protocol === 'SPP') {
					this.sendSPPData(dataString);
				} else {
					this.sendBLEData(dataString);
				}

				// 每发完一行（例如宏指令），给单片机预留 200 毫秒消化时间，再发下一行（如 S@）
				if (i < this.commandQueue.length - 1) {
					await new Promise(resolve => setTimeout(resolve, 200));
				}
			}
        },

        syncToChatCache(type, content) {
            const cacheKey = 'chat_history_' + this.deviceId;
            let history = uni.getStorageSync(cacheKey) || [];
            const now = new Date();
            const timeStr = `${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}:${now.getSeconds().toString().padStart(2, '0')}`;
            history.push({ type: type, text: '控制指令: ' + content, time: timeStr });
            uni.setStorageSync(cacheKey, history);
            uni.$emit('refreshChat');
        },

        initBLEService() {
            uni.getBLEDeviceServices({
                deviceId: this.deviceId,
                success: (res) => {
                    const mainService = res.services.find(s => s.uuid.includes('FFE0') || s.uuid.includes('FFF0')) || res.services[0];
                    this.serviceId = mainService.uuid;
                    uni.getBLEDeviceCharacteristics({
                        deviceId: this.deviceId, serviceId: this.serviceId,
                        success: (cRes) => {
                            const char = cRes.characteristics.find(c => c.properties.write);
                            if (char) this.characteristicId = char.uuid;
                        }
                    });
                }
            });
        },

		// 核心优化：智能处理预设的还原 和 自动加 @ 符号
        addCommand(keyOrCode, name, isCustom = false) { 
			let code = isCustom ? keyOrCode : this.config[keyOrCode];
			
			if (isCustom) {
				// 通过 @ 切割之前粘连保存的字符串，将其还原成多个指令数组
				let parts = code.split('@').filter(p => p.trim() !== '');
				
				if (parts.length > 1) {
					parts.forEach((p, idx) => {
						let c = p + '@'; // 给切出来的每段重新补上 @
						let partName = name;
						if (idx === parts.length - 1 && p === (this.config.start || 'S')) {
							partName = '自动触发启动';
						} else {
							partName = idx === 0 ? name + ' (路线)' : name + ` (部分${idx+1})`;
						}
						this.commandQueue.push({ name: partName, code: c });
					});
					this.saveQueueState();
					return;
				}
			}

			// 如果是单个小指令，防漏补上 @
			if (!code.endsWith('@')) {
				code += '@';
			}
			
			this.commandQueue.push({ name, code }); 
			this.saveQueueState(); 
		},
		
        deleteLast() { 
			this.commandQueue.pop(); 
			this.saveQueueState(); 
		},
        saveConfig() { uni.setStorageSync('bt_char_config_track1', this.config); this.showConfig = false; uni.showToast({ title: '配置已保存' }); },
        
		goToChat() { 
			if (!this.isConnected) uni.showToast({ title: '未连接蓝牙，当前为离线预览模式', icon: 'none' });
			uni.navigateTo({ url: `/pages/dueihua/dueihua?deviceName=${encodeURIComponent(this.deviceName)}&protocol=${this.protocol}&deviceId=${this.deviceId}` }); 
		},
		
		onStartPointChange(e) { this.startPointIndex = parseInt(e.detail.value); },
		
		generatePathLines(path, startLine) {
		    let lines = [];
		    let currentX = startLine * 20;
		    let currentY = 100;

		    for (let i = 0; i < path.length; i++) {
		        let targetCol = path[i];
		        let targetX = targetCol * 20 + 10;
		        
		        if (currentX !== targetX) {
		            let hLeft = Math.min(currentX, targetX);
		            let hWidth = Math.abs(currentX - targetX);
		            lines.push({
		                left: `calc(${hLeft}% - 5rpx)`,
		                top: `calc(${currentY}% - 5rpx)`,
		                width: `calc(${hWidth}% + 10rpx)`,
		                height: '10rpx'
		            });
		        }
		        
		        currentX = targetX;

		        let nextY = currentY - 20;
		        lines.push({
		            left: `calc(${currentX}% - 5rpx)`,
		            top: `calc(${nextY}% - 5rpx)`,
		            width: '10rpx',
		            height: `calc(20% + 10rpx)`
		        });
		        
		        currentY = nextY;
		    }

		    lines.push({
		        left: `calc(${currentX}% - 5rpx)`,
		        top: `calc(${currentY - 12}% - 5rpx)`,
		        width: '10rpx',
		        height: `calc(12% + 10rpx)`
		    });
		    
		    let startX = startLine * 20;
		    lines.push({
		        left: `calc(${startX}% - 5rpx)`,
		        top: `calc(100% - 5rpx)`,
		        width: '10rpx',
		        height: `calc(12% + 10rpx)`
		    });

		    this.pathLines = lines;
		}
    }
}
</script>

<style>
/* 基础容器 */
.container { background-color: #f5f7f9; min-height: 100vh; }
.status-bar { height: var(--status-bar-height); background-color: #ffffff; }
.safe-bottom { height: 40rpx; }

.track-switcher { display: flex; margin: 20rpx 24rpx; background: #e2e8f0; border-radius: 16rpx; padding: 6rpx; }
.track-btn { flex: 1; text-align: center; padding: 18rpx 0; font-size: 28rpx; font-weight: bold; color: #64748b; border-radius: 12rpx; transition: 0.3s; }
.track-btn.active { background: #ffffff; color: #3b82f6; box-shadow: 0 4rpx 10rpx rgba(0,0,0,0.06); }

.connection-header { display: flex; justify-content: space-between; align-items: center; background-color: #ffffff; padding: 24rpx 30rpx; border-bottom: 1rpx solid #eeeeee; position: sticky; top: 0; z-index: 100; }
.btn-config-entry { margin: 0; font-size: 22rpx; background: #f1f5f9; color: #475569; height: 54rpx; line-height: 54rpx; padding: 0 20rpx; }
.conn-status { display: flex; align-items: center; background-color: #f0f9ff; padding: 12rpx 24rpx; border-radius: 40rpx; width: fit-content; }
.status-dot { width: 16rpx; height: 16rpx; border-radius: 50%; background-color: #94a3b8; margin-right: 16rpx; }
.status-dot.active { background-color: #22c55e; box-shadow: 0 0 10rpx rgba(34, 197, 94, 0.4); }
.conn-text { font-size: 26rpx; color: #0369a1; font-weight: 600; }
.toggle-hint { font-size: 20rpx; color: #94a3b8; margin-left: 10rpx; font-weight: normal; }

.scan-panel { background: #ffffff; border-bottom: 1rpx solid #eeeeee; box-shadow: 0 4rpx 12rpx rgba(0,0,0,0.02); }
.scan-controls { display: flex; align-items: center; justify-content: space-between; padding: 20rpx 30rpx; }
.btn-scan { background: linear-gradient(135deg, #007AFF, #0055FF); color: #fff; font-size: 26rpx; border-radius: 50rpx; margin: 0; padding: 0 40rpx; height: 60rpx; line-height: 60rpx; flex-shrink: 0;}
.filter-tag { font-size: 24rpx; color: #555; background: #f1f5f9; padding: 10rpx 24rpx; border-radius: 30rpx; border: 1rpx solid #e2e8f0; }
.btn-disconnect-sm { background: #fee2e2; color: #ef4444; font-size: 22rpx; border-radius: 30rpx; padding: 0 24rpx; height: 50rpx; line-height: 50rpx; margin: 0; font-weight: bold;}

.device-list-container { max-height: 450rpx; padding: 0 20rpx; box-sizing: border-box; background: #f8fafc; border-top: 1rpx dashed #e2e8f0; }
.empty-box { text-align: center; padding: 40rpx 0; }
.empty-text { color: #94a3b8; font-size: 24rpx; }

.device-item { background: #ffffff; margin: 15rpx 0; padding: 24rpx; border-radius: 16rpx; display: flex; justify-content: space-between; align-items: center; border: 1rpx solid #e2e8f0; }
.device-name { font-size: 28rpx; font-weight: 600; color: #334155; }
.device-mac { font-size: 20rpx; color: #94a3b8; margin-top: 6rpx; display: block; font-family: monospace; }
.tags { display: flex; margin-top: 12rpx; align-items: center; }
.tag-protocol { font-size: 16rpx; padding: 4rpx 10rpx; border-radius: 6rpx; margin-right: 12rpx; font-weight: bold; }
.tag-protocol.BLE { background: #E3F2FD; color: #1976D2; }
.tag-protocol.SPP { background: #F3E5F5; color: #7B1FA2; }

.tag-rssi { font-size: 18rpx; }
.rssi-good { color: #10b981; font-weight: bold; }
.rssi-normal { color: #f59e0b; font-weight: bold; }
.rssi-bad { color: #ef4444; font-weight: bold; }
.rssi-unknown { color: #94a3b8; }

.tag-connected { background: #dcfce7; color: #16a34a; font-size: 16rpx; padding: 4rpx 10rpx; border-radius: 6rpx; margin-left: 10rpx; font-weight: bold; }
.btn-connect { font-size: 24rpx; color: #ffffff; background: #3b82f6; border-radius: 30rpx; margin: 0; padding: 0 30rpx; height: 56rpx; line-height: 56rpx; font-weight: bold;}
.btn-connect[disabled] { background: #f1f5f9 !important; color: #94a3b8 !important; }

/* 控制面板样式 */
.card { background: #ffffff; margin: 24rpx; padding: 32rpx; border-radius: 28rpx; box-shadow: 0 6rpx 24rpx rgba(0,0,0,0.04); }
.card-header { display: flex; align-items: center; margin-bottom: 24rpx; }
.card-header-between { display: flex; justify-content: space-between; align-items: center; margin-bottom: 24rpx; }
.card-title { font-size: 34rpx; font-weight: 800; color: #1e293b; }
.card-title.block { display: block; margin-bottom: 20rpx; }
.header-icon { font-size: 38rpx; margin-right: 12rpx; color: #1e293b; }
.text-link { color: #ef4444; font-size: 26rpx; font-weight: 600; }
.text-blue { color: #3b82f6; font-weight: 600; }

button { margin: 0; padding: 0; border: none; border-radius: 16rpx; }
button::after { border: none; }

.btn-main-action { background: linear-gradient(to right, #6366f1, #3b82f6); color: #ffffff; font-weight: bold; font-size: 30rpx; height: 100rpx; line-height: 100rpx; margin-bottom: 30rpx; }
.grid-btn { background-color: #e2e8f0; color: #64748b; font-size: 26rpx; font-weight: 600; height: 84rpx; line-height: 84rpx; transition: background-color 0.1s; }
.grid-btn:active { background-color: #3b82f6 !important; color: #ffffff !important; }
.custom-btn { border: 1rpx solid #cbd5e1; background-color: #f1f5f9; color: #3b82f6; }

.btn-send-active { background: linear-gradient(to right, #10b981, #3b82f6); color: #ffffff; height: 100rpx; line-height: 100rpx; font-weight: bold; box-shadow: 0 4rpx 12rpx rgba(59, 130, 246, 0.3); }
.btn-delete { background-color: #fee2e2; color: #ef4444; height: 90rpx; line-height: 90rpx; font-size: 28rpx; margin-top: 10rpx; font-weight: 600; }

.section { padding: 24rpx; border-radius: 20rpx; margin-bottom: 20rpx; }
.blue-bg { background-color: #f0f7ff; }
.cyan-bg { background-color: #ecfeff; }
.purple-bg { background-color: #faf5ff; }
.orange-bg { background-color: #fff7ed; }
.red-bg { background-color: #fef2f2; border: 1px dashed #fca5a5; }

.section-title { font-size: 28rpx; font-weight: bold; color: #334155; display: flex; align-items: center; }
.title-arrow, .title-icon { margin-right: 10rpx; font-size: 32rpx; }
.button-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16rpx; margin-top: 20rpx;}
.button-grid-3 { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16rpx; margin-top: 20rpx;}
.button-grid-2 { display: grid; grid-template-columns: repeat(2, 1fr); gap: 16rpx; margin-top: 20rpx;}
.code-hint { font-size: 20rpx; opacity: 0.6; margin-left: 8rpx; font-weight: normal; }
.queue-display-area { min-height: 160rpx; background: #f8fafc; border-radius: 16rpx; padding: 20rpx; margin-bottom: 20rpx; }
.queue-list { display: flex; flex-wrap: wrap; gap: 10rpx; }
.queue-item { background: #3b82f6; color: #fff; padding: 6rpx 16rpx; border-radius: 8rpx; font-size: 24rpx; }
.empty-placeholder { padding: 40rpx 0; text-align: center; }
.empty-text { font-size: 32rpx; color: #94a3b8; font-weight: 600; margin-bottom: 8rpx; }
.empty-subtext { font-size: 24rpx; color: #cbd5e1; }

/* 双模式切换样式 */
.mode-tabs { display: flex; background: #f1f5f9; border-radius: 12rpx; padding: 8rpx; margin-bottom: 24rpx; }
.mode-tab { flex: 1; text-align: center; font-size: 28rpx; color: #64748b; padding: 16rpx 0; border-radius: 8rpx; transition: all 0.3s; font-weight: bold; }
.mode-tab.active { background: #ffffff; color: #3b82f6; box-shadow: 0 4rpx 10rpx rgba(0,0,0,0.06); }
.manual-hint { font-size: 24rpx; color: #f59e0b; text-align: center; margin-top: 10rpx; margin-bottom: 20rpx; font-weight: bold; }
.manual-btn { background: #fef3c7 !important; color: #d97706 !important; border-color: #fde68a !important; margin-top: 10rpx; }
.manual-btn:active { background: #fde68a !important; }

/* === 运动地图样式 === */
.map-container { border: 4rpx solid #333; border-radius: 8rpx; overflow: visible; }
.map-header-row, .map-footer-row { display: flex; height: 60rpx; }
.row-label-empty { width: 100rpx; background-color: #fff; border-right: 4rpx solid #333; }

.end-point-bar, .start-zone-bar { flex: 1; position: relative; display: flex; align-items: center; }
.end-point-bar { background-color: #7ef27e; border-bottom: 4rpx solid #333; }
.start-zone-bar { background-color: #ff9999; border-top: none; border-bottom: 4rpx solid #333; } 

.v-line { position: absolute; top: 0; bottom: 0; width: 4rpx; background-color: #333; z-index: 1; }
.v-line:nth-child(1) { left: 20%; transform: translateX(-2rpx); }
.v-line:nth-child(2) { left: 40%; transform: translateX(-2rpx); }
.v-line:nth-child(3) { left: 60%; transform: translateX(-2rpx); }
.v-line:nth-child(4) { left: 80%; transform: translateX(-2rpx); }

.bar-text { position: relative; z-index: 10; display: flex; width: 100%; height: 100%; }
.bar-text text { flex: 1; display: flex; align-items: center; justify-content: center; font-weight: bold; font-size: 24rpx; color: #333; }

.map-axis-row { display: flex; height: 51rpx; }
.axis-content { flex: 1; position: relative; }
.v-line-ext { position: absolute; top: 0; width: 4rpx; height: 53rpx; background-color: #333; }
.v-line-ext:nth-child(1) { left: 0; display: none; } 
.v-line-ext:nth-child(2) { left: 20%; transform: translateX(-2rpx); }
.v-line-ext:nth-child(3) { left: 40%; transform: translateX(-2rpx); }
.v-line-ext:nth-child(4) { left: 60%; transform: translateX(-2rpx); }
.v-line-ext:nth-child(5) { left: 80%; transform: translateX(-2rpx); }
.v-line-ext:nth-child(6) { right: 0; display: none; }

.axis-numbers { display: flex; width: 100%; height: 100%; }
.axis-numbers text { position: absolute; top: 63rpx; font-size: 20rpx; font-weight: bold; color: #333; transform: translateX(-50%); }
.axis-numbers text:nth-child(1) { left: 0; }
.axis-numbers text:nth-child(2) { left: 20%; }
.axis-numbers text:nth-child(3) { left: 40%; }
.axis-numbers text:nth-child(4) { left: 60%; }
.axis-numbers text:nth-child(5) { left: 80%; }
.axis-numbers text:nth-child(6) { left: 100%; }

.map-body { border: none; position: relative; }
.grid-row { display: flex;}
.grid-row:last-child { border-bottom: none; }

.row-side-label { width: 100rpx; font-size: 16rpx; color: #333; display: flex; align-items: center; justify-content: center; writing-mode: vertical-lr; border-right: 4rpx solid #333; }

.grid-cell { flex: 1; height: 100rpx; background-color: #fff; border-right: 4rpx solid #333; border-bottom: 4rpx solid #333; position: relative; display: flex; align-items: center; justify-content: center; transition: all 0.3s ease; }
.grid-cell:last-child { border-right: none; }
.grid-cell.path-active { background-color: #ffe4e1; }

.cell-num { position: absolute; top: 4rpx; left: 8rpx; font-size: 20rpx; color: #333; }

.nested-box { display: flex; align-items: center; justify-content: center; }
.box-red { width: 50rpx; height: 50rpx; background-color: #ff0000; }
.box-yellow { width: 30rpx; height: 30rpx; background-color: #ffff00; }
.box-green { width: 14rpx; height: 14rpx; background-color: #00ff00; }

.path-overlay { position: absolute; top: 0; bottom: 0; left: 104rpx; right: 0; z-index: 10; pointer-events: none; }
.path-line { position: absolute; background-color: #ef4444; border-radius: 6rpx; box-shadow: 0 0 6rpx rgba(239, 68, 68, 0.6); transition: all 0.5s ease; }

/* === 总体配置与自定义指令模块样式 === */
.config-mask { position: fixed; top: 0; left: 0; right: 0; bottom: 0; background: rgba(0,0,0,0.6); z-index: 9999; display: flex; align-items: center; justify-content: center; }
.config-panel { width: 85%; background: #ffffff; border-radius: 30rpx; padding: 40rpx; height: 800rpx; display: flex; flex-direction: column; }
.config-title { font-weight: bold; margin-bottom: 10rpx; text-align: center; color: #1e293b; }
.config-scroll { flex: 1; height: 1px; overflow-y: scroll; padding: 10rpx 0; }
.config-collapse-header { display: flex; justify-content: space-between; align-items: center; background: #f1f5f9; padding: 20rpx 30rpx; border-radius: 12rpx; margin-top: 16rpx; font-size: 28rpx; color: #334155; font-weight: bold; }
.config-collapse-body { background: #ffffff; padding: 10rpx 20rpx; border: 1px solid #f1f5f9; border-top: none; border-radius: 0 0 12rpx 12rpx; }
.config-item { display: flex; align-items: center; justify-content: space-between; margin-bottom: 20rpx; border-bottom: 1rpx dashed #f1f5f9; padding-bottom: 12rpx; padding-top: 10rpx;}
.config-item text { font-size: 26rpx; color: #64748b; }
.config-item input { width: 160rpx; background: #f8fafc; text-align: center; font-size: 26rpx; border-radius: 10rpx; padding: 10rpx; border: 1rpx solid #e2e8f0; }

.custom-cmd-form { background: #f8fafc; padding: 20rpx; border-radius: 12rpx; margin-bottom: 16rpx; margin-top: 10rpx;}
.custom-picker { font-size: 26rpx; color: #475569; margin-bottom: 20rpx; border-bottom: 1rpx solid #e2e8f0; padding-bottom: 10rpx; }
.custom-input-row { display: flex; gap: 16rpx; margin-bottom: 20rpx; }
.c-input { flex: 1; background: #fff; border: 1rpx solid #cbd5e1; border-radius: 8rpx; padding: 12rpx; font-size: 24rpx; text-align: center; }
.btn-add-custom { background: #10b981; color: #fff; font-size: 26rpx; height: 64rpx; line-height: 64rpx; border-radius: 8rpx; font-weight: bold;}
.custom-list { margin-top: 10rpx; }
.custom-mod-title { font-size: 24rpx; color: #94a3b8; margin-top: 16rpx; margin-bottom: 10rpx; font-weight: bold;}
.custom-cmd-row { display: flex; justify-content: space-between; align-items: center; background: #f1f5f9; padding: 16rpx 20rpx; border-radius: 8rpx; margin-bottom: 10rpx; }
.cc-name { font-size: 24rpx; color: #334155; font-weight: 500;}
.cc-del { font-size: 24rpx; color: #ef4444; font-weight: bold;}

.config-footer { display: flex; gap: 20rpx; margin-top: 20rpx; }
.btn-save-config { flex: 2; background: #3b82f6; color: #fff; height: 84rpx; line-height: 84rpx; border-radius: 42rpx; font-weight: bold; }
.btn-exit-config { flex: 1; background: #f1f5f9; color: #64748b; height: 84rpx; line-height: 84rpx; border-radius: 42rpx; font-size: 26rpx; }

.safe-bottom { height: 120rpx; }

.chat-preview-box { display: flex; justify-content: space-between; align-items: center; background-color: #f0fdf4; border: 2rpx dashed #22c55e; padding: 20rpx 30rpx; border-radius: 20rpx; margin-bottom: 30rpx; }
.chat-content { display: flex; align-items: center; }
.chat-icon { font-size: 40rpx; margin-right: 20rpx; }
.chat-text-area { display: flex; flex-direction: column; }
.chat-label { font-size: 28rpx; font-weight: bold; color: #166534; }
.chat-hint { font-size: 22rpx; color: #15803d; margin-top: 4rpx; }

.path-planning-box { padding: 10rpx 0; }
.input-group { display: flex; justify-content: space-between; align-items: center; background: #f8fafc; padding: 20rpx; border-radius: 12rpx; margin-bottom: 16rpx; }
.input-label { font-size: 28rpx; color: #475569; font-weight: 500; }
.picker-display { font-size: 28rpx; color: #3b82f6; font-weight: bold; }
.sum-input { text-align: right; font-size: 28rpx; width: 300rpx; color: #1e293b; font-weight: bold; border-bottom: 1px solid #cbd5e1; }
.btn-generate-path { background: #e0e7ff; color: #4338ca; height: 88rpx; line-height: 88rpx; border-radius: 16rpx; font-size: 28rpx; font-weight: bold; margin-top: 20rpx; border: 1px solid #c7d2fe; }
.btn-generate-path:active { background: #c7d2fe; }
.path-result { margin-top: 24rpx; background: #f0fdf4; border-radius: 12rpx; padding: 20rpx; border: 1px dashed #86efac; }
.result-row { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12rpx; }
.result-row:last-child { margin-bottom: 0; }
.result-tag { font-size: 24rpx; color: #166534; }
.result-val { font-size: 28rpx; font-weight: bold; color: #15803d; }
</style>