<template>
	<scroll-view :scroll-y="!showConfig" class="container">
		<view class="status-bar"></view>

		<view class="track-switcher">
			<view class="track-btn" @tap="goToTrack1">🏁 赛道一 (基础巡线)</view>
			<view class="track-btn active">🚗 赛道二 (算术巡航)</view>
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

				<view class="device-item" v-for="item in filteredDeviceList" :key="item.deviceId">
					<view class="device-info">
						<text class="device-name">{{ item.name || 'Unknown Device' }}</text>
						<text class="device-mac">{{ item.deviceId }}</text>
						<view class="tags">
							<text class="tag-protocol" :class="item.protocol">{{ item.protocol }}</text>
							<text class="tag-rssi" :class="getRssiClass(item.RSSI)">📶 {{ item.RSSI }}{{ item.RSSI !== 'Unknown' ? 'dBm' : '' }}</text>
							<text v-if="deviceId === item.deviceId" class="tag-connected">已连接</text>
						</view>
					</view>
					<button
						class="btn-connect"
						:loading="connectingId === item.deviceId"
						:disabled="deviceId === item.deviceId"
						@tap.stop="connectTo(item)"
					>
						{{ deviceId === item.deviceId ? '已连' : '连接' }}
					</button>
				</view>
			</scroll-view>
		</view>

		<view class="card">
			<view class="card-header">
				<text class="header-icon">⏻</text>
				<text class="card-title">赛道二控制面板</text>
			</view>

			<button class="btn-main-action" @tap="addCommand('start', '启动运行')">
				<text class="play-icon">▶</text>
				启动运行
				<text class="code-hint">({{ config.start }})</text>
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
				<view class="section-title">
					<text class="title-icon">🌟</text> 我的预设路线
				</view>
				<view class="button-grid-2">
					<button
						class="grid-btn custom-btn"
						v-for="(cmd, idx) in customCommands.routes"
						:key="'croute'+idx"
						@tap="addCommand(cmd.code, cmd.name, true)"
					>
						{{ cmd.name }}
					</button>
				</view>
			</view>

			<view class="section blue-bg">
				<view class="section-title">
					<text class="title-arrow">↑</text> 直走到路口
				</view>
				<view class="button-grid">
					<button v-for="i in 5" :key="'go'+i" class="grid-btn" @tap="addCommand('go' + i, '直走' + i)">
						↑ 直走{{ i }} ({{ config['go' + i] }})
					</button>
					<button
						class="grid-btn custom-btn"
						v-for="(cmd, idx) in customCommands.lines"
						:key="'clines'+idx"
						@tap="addCommand(cmd.code, cmd.name, true)"
					>
						{{ cmd.name }} ({{ cmd.code }})
					</button>
				</view>
			</view>

			<view class="section cyan-bg">
				<view class="section-title">
					<text class="title-icon">🕒</text> 直走时间
				</view>
				<view class="button-grid-3">
					<button v-for="i in 3" :key="'time'+i" class="grid-btn" @tap="addCommand('time' + i, '直走' + i + '秒')">
						{{ i }}秒 ({{ config['time' + i] }})
					</button>
					<button
						class="grid-btn custom-btn"
						v-for="(cmd, idx) in customCommands.time"
						:key="'ctime'+idx"
						@tap="addCommand(cmd.code, cmd.name, true)"
					>
						{{ cmd.name }} ({{ cmd.code }})
					</button>
				</view>
			</view>

			<view class="section purple-bg">
				<view class="section-title">操作指令</view>
				<view class="button-grid-2">
					<button class="grid-btn" @tap="addCommand('left', '左拐弯')">← 左拐弯 ({{ config.left }})</button>
					<button class="grid-btn" @tap="addCommand('right', '右拐弯')">→ 右拐弯 ({{ config.right }})</button>
					<button class="grid-btn" @tap="addCommand('stop', '停止')">🔲 停止 ({{ config.stop }})</button>
					<button class="grid-btn" @tap="addCommand('light', '亮灯')">💡 亮灯 ({{ config.light }})</button>
					<button
						class="grid-btn custom-btn"
						v-for="(cmd, idx) in customCommands.base"
						:key="'cbase'+idx"
						@tap="addCommand(cmd.code, cmd.name, true)"
					>
						{{ cmd.name }} ({{ cmd.code }})
					</button>
				</view>
			</view>

			<view class="section orange-bg">
				<view class="section-title">
					<text class="title-icon">📍</text> 路径规划专用动作集
				</view>
				<view class="button-grid-3">
					<button class="grid-btn" @tap="addCommand('crossLeft', '左转(横向移动)')">横向左转 ({{ config.crossLeft }})</button>
					<button class="grid-btn" @tap="addCommand('crossRight', '右转(横向移动)')">横向右转 ({{ config.crossRight }})</button>
					<button class="grid-btn" @tap="addCommand('findCol', '寻线至正下方')">寻列下方 ({{ config.findCol }})</button>

					<button class="grid-btn" @tap="addCommand('alignLeft', '左转回正(面向上)')">左转回正 ({{ config.alignLeft }})</button>
					<button class="grid-btn" @tap="addCommand('alignRight', '右转回正(面向上)')">右转回正 ({{ config.alignRight }})</button>
					<button class="grid-btn" @tap="addCommand('enterBlock', '直行进入方块中心')">进块中心 ({{ config.enterBlock }})</button>

					<button class="grid-btn" @tap="addCommand('pauseBlock', '中心停留1秒')">停留1秒 ({{ config.pauseBlock }})</button>
					<button class="grid-btn" @tap="addCommand('exitBlock', '直行穿出到上方路口')">直穿出块 ({{ config.exitBlock }})</button>
					<button class="grid-btn" @tap="addCommand('goFinish', '直行进入终点区')">进终点区 ({{ config.goFinish }})</button>

					<button
						class="grid-btn custom-btn"
						v-for="(cmd, idx) in customCommands.advanced"
						:key="'cadv'+idx"
						@tap="addCommand(cmd.code, cmd.name, true)"
					>
						{{ cmd.name }} ({{ cmd.code }})
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
				<text class="card-title">赛道二智能规划</text>
			</view>

			<view class="path-planning-box">
				<view class="mode-tabs">
					<view class="mode-tab" :class="{ active: planMode === 'auto' }" @tap="switchMode('auto')">🤖 自动匹配</view>
					<view class="mode-tab" :class="{ active: planMode === 'manual' }" @tap="switchMode('manual')">👆 手动选择</view>
				</view>

				<view class="input-group">
					<text class="input-label">当前目标结果</text>
					<text class="picker-display text-blue">{{ selectedResult || '请点顶部结果区' }}</text>
				</view>

				<view class="input-group">
					<text class="input-label">当前公式</text>
					<text class="picker-display text-blue">{{ manualResultText }}</text>
				</view>

				<view class="manual-hint">
					💡 先点三层数字；自动模式会自动补两层运算符，手动模式需要你自己点两层运算符
				</view>

				<button class="btn-generate-path" @tap="calculateTrack2Path">一键生成赛道二路径</button>

				<button v-if="planMode === 'manual'" class="btn-generate-path manual-btn" @tap="generateManualPathCmd">
					生成手动指令队列
				</button>

				<view v-if="bestPathObj" class="path-result">
					<view class="result-row">
						<text class="result-tag">最优公式</text>
						<text class="result-val">{{ bestPathObj.equation }}</text>
					</view>
					<view class="result-row">
						<text class="result-tag">路径横移代价</text>
						<text class="result-val">{{ bestPathObj.distance }}</text>
					</view>
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
						<text class="q-code" v-if="item.code.length < 20">[{{ item.code }}]</text>
					</view>
				</view>
			</view>

			<button class="btn-send-active" @tap="sendBatch">
				<text class="send-icon">🚀</text> 发送指令 ({{ commandQueue.length }})
			</button>
		</view>

		<view class="card">
			<view class="card-header-between">
				<text class="card-title">赛道二运动地图</text>
				<text class="text-link" @tap="clearMap">🗑️ 重新画线</text>
			</view>

			<view class="track2-photo-map">
				<svg class="track2-svg" viewBox="0 0 100 100" preserveAspectRatio="none" v-if="svgPoints">
					<polyline
						:points="svgPoints"
						fill="none"
						stroke="#ff3b30"
						stroke-width="1.35"
						stroke-linecap="round"
						stroke-linejoin="round"
						stroke-dasharray="2.6,2.2"
					/>
				</svg>

				<view class="track2-row track2-top-row">
					<view class="track2-side-finish left">终</view>

					<view
						v-for="item in resultCells"
						:key="'result-' + item"
						class="track2-result-cell"
						:class="{ active: isResultActive(item) }"
						@tap="onResultTap(item)"
					>
						<text class="track2-result-num">{{ item }}</text>
						<text class="track2-result-mark">||</text>
					</view>

					<view class="track2-side-finish right">点</view>
				</view>

				<view class="track2-row track2-number-row">
					<view class="track2-side-blank"></view>
					<view
						v-for="(n, colIndex) in numberRows[0]"
						:key="'n1-' + colIndex"
						class="track2-number-cell"
						:class="{ active: isNumberActive(0, n) }"
						@tap="onNumberTap(0, n, colIndex)"
					>
						<text>{{ n }}</text>
					</view>
					<view class="track2-side-blank"></view>
				</view>

				<view class="track2-op-wrapper">
					<svg class="track2-op-svg" viewBox="0 0 620 100" preserveAspectRatio="none" style="width: 100%; height: 100%; position: absolute; top:0; left:0; z-index: 1; display: block;">
						<path class="lane-path plus" :class="{ active: isOperatorActive(0, '+') }" 
							  d="M 60,0 C -30,30 150,70 60,100 L 160,100 C 250,70 70,30 160,0 Z" />
						<path class="lane-path minus" :class="{ active: isOperatorActive(0, '-') }" 
							  d="M 160,0 C 70,30 250,70 160,100 L 260,100 C 350,70 170,30 260,0 Z" />
						<path class="lane-path multiply" :class="{ active: isOperatorActive(0, '×') }" 
							  d="M 360,0 C 270,30 450,70 360,100 L 460,100 C 550,70 370,30 460,0 Z" />
						<path class="lane-path divide" :class="{ active: isOperatorActive(0, '÷') }" 
							  d="M 460,0 C 370,30 550,70 460,100 L 560,100 C 650,70 470,30 560,0 Z" />
					</svg>
					
					<view class="track2-row h-full" style="position: relative; z-index: 5;">
						<view class="track2-side-blank"></view>
						<view class="track2-op-text-zone" @tap="onOperatorTap(0, '+')">
							<text class="top-op">+</text><text class="mid-op">+</text><text class="bot-op">+</text>
						</view>
						<view class="track2-op-text-zone minus" @tap="onOperatorTap(0, '-')">
							<text class="top-op">−</text><text class="mid-op">−</text><text class="bot-op">−</text>
						</view>
						<view class="track2-op-empty-zone"></view>
						<view class="track2-op-text-zone" @tap="onOperatorTap(0, '×')">
							<text class="top-op">×</text><text class="mid-op">×</text><text class="bot-op">×</text>
						</view>
						<view class="track2-op-text-zone" @tap="onOperatorTap(0, '÷')">
							<text class="top-op">÷</text><text class="mid-op">÷</text><text class="bot-op">÷</text>
						</view>
						<view class="track2-side-blank"></view>
					</view>
				</view>

				<view class="track2-row track2-number-row">
					<view class="track2-side-blank"></view>
					<view
						v-for="(n, colIndex) in numberRows[1]"
						:key="'n2-' + colIndex"
						class="track2-number-cell"
						:class="{ active: isNumberActive(1, n) }"
						@tap="onNumberTap(1, n, colIndex)"
					>
						<text>{{ n }}</text>
					</view>
					<view class="track2-side-blank"></view>
				</view>

				<view class="track2-op-wrapper">
					<svg class="track2-op-svg" viewBox="0 0 620 100" preserveAspectRatio="none" style="width: 100%; height: 100%; position: absolute; top:0; left:0; z-index: 1; display: block;">
						<path class="lane-path plus" :class="{ active: isOperatorActive(1, '+') }" 
							  d="M 60,0 C -30,30 150,70 60,100 L 160,100 C 250,70 70,30 160,0 Z" />
						<path class="lane-path minus" :class="{ active: isOperatorActive(1, '-') }" 
							  d="M 160,0 C 70,30 250,70 160,100 L 260,100 C 350,70 170,30 260,0 Z" />
						<path class="lane-path multiply" :class="{ active: isOperatorActive(1, '×') }" 
							  d="M 360,0 C 270,30 450,70 360,100 L 460,100 C 550,70 370,30 460,0 Z" />
						<path class="lane-path divide" :class="{ active: isOperatorActive(1, '÷') }" 
							  d="M 460,0 C 370,30 550,70 460,100 L 560,100 C 650,70 470,30 560,0 Z" />
					</svg>

					<view class="track2-row h-full" style="position: relative; z-index: 5;">
						<view class="track2-side-blank"></view>
						<view class="track2-op-text-zone" @tap="onOperatorTap(1, '+')">
							<text class="top-op">+</text><text class="mid-op">+</text><text class="bot-op">+</text>
						</view>
						<view class="track2-op-text-zone minus" @tap="onOperatorTap(1, '-')">
							<text class="top-op">−</text><text class="mid-op">−</text><text class="bot-op">−</text>
						</view>
						<view class="track2-op-empty-zone"></view>
						<view class="track2-op-text-zone" @tap="onOperatorTap(1, '×')">
							<text class="top-op">×</text><text class="mid-op">×</text><text class="bot-op">×</text>
						</view>
						<view class="track2-op-text-zone" @tap="onOperatorTap(1, '÷')">
							<text class="top-op">÷</text><text class="mid-op">÷</text><text class="bot-op">÷</text>
						</view>
						<view class="track2-side-blank"></view>
					</view>
				</view>

				<view class="track2-row track2-number-row">
					<view class="track2-side-blank"></view>
					<view
						v-for="(n, colIndex) in numberRows[2]"
						:key="'n3-' + colIndex"
						class="track2-number-cell"
						:class="{ active: isNumberActive(2, n) }"
						@tap="onNumberTap(2, n, colIndex)"
					>
						<text>{{ n }}</text>
					</view>
					<view class="track2-side-blank"></view>
				</view>

				<view class="track2-row track2-start-row">
					<view class="track2-start-cell"></view>
					<view class="track2-start-cell text-cell">启</view>
					<view class="track2-start-cell"></view>
					<view class="track2-start-cell text-cell">动</view>
					<view class="track2-start-cell"></view>
					<view class="track2-start-cell text-cell">区</view>
					<view class="track2-start-cell"></view>
				</view>
			</view>
		</view>

		<view class="config-mask" v-if="showConfig" @tap.self="showConfig = false" @touchmove.stop.prevent>
			<view class="config-panel" @touchmove.stop>
				<view class="config-title">总体配置 - 设置发送字符</view>

				<scroll-view scroll-y="true" class="config-scroll">
					<view class="config-collapse-header" @tap="toggleConfigTab('custom')">
						<text>🛠️ 自定义指令管理</text>
						<text>{{ configTabs.custom ? '▲' : '▼' }}</text>
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
							<view
								class="custom-list-item"
								v-for="(modName, modKey) in moduleNamesMap"
								:key="modKey"
								v-if="customCommands[modKey] && customCommands[modKey].length > 0"
							>
								<view class="custom-mod-title">【{{ modName }}】：</view>
								<view class="custom-cmd-row" v-for="(cmd, cIdx) in customCommands[modKey]" :key="cIdx">
									<text class="cc-name">{{ cmd.name }}</text>
									<text class="cc-del" @tap="removeCustomCommand(modKey, cIdx)">❌ 删除</text>
								</view>
							</view>
						</view>
					</view>

					<view class="config-collapse-header" @tap="toggleConfigTab('base')">
						<text>🕹️ 基础运动指令</text>
						<text>{{ configTabs.base ? '▲' : '▼' }}</text>
					</view>
					<view class="config-collapse-body" v-show="configTabs.base">
						<view class="config-item"><text>启动运行</text><input v-model="config.start" /></view>
						<view class="config-item"><text>左拐弯</text><input v-model="config.left" /></view>
						<view class="config-item"><text>右拐弯</text><input v-model="config.right" /></view>
						<view class="config-item"><text>停止</text><input v-model="config.stop" /></view>
						<view class="config-item"><text>亮灯</text><input v-model="config.light" /></view>
					</view>

					<view class="config-collapse-header" @tap="toggleConfigTab('lines')">
						<text>📏 直走跨线指令</text>
						<text>{{ configTabs.lines ? '▲' : '▼' }}</text>
					</view>
					<view class="config-collapse-body" v-show="configTabs.lines">
						<view class="config-item" v-for="i in 5" :key="'cgo'+i">
							<text>直走{{ i }}</text><input v-model="config['go' + i]" />
						</view>
					</view>

					<view class="config-collapse-header" @tap="toggleConfigTab('time')">
						<text>🕒 直走延时指令</text>
						<text>{{ configTabs.time ? '▲' : '▼' }}</text>
					</view>
					<view class="config-collapse-body" v-show="configTabs.time">
						<view class="config-item" v-for="i in 3" :key="'ctm'+i">
							<text>直走{{ i }}秒</text><input v-model="config['time' + i]" />
						</view>
					</view>

					<view class="config-collapse-header" @tap="toggleConfigTab('advanced')">
						<text>📍 高级专用动作代码</text>
						<text>{{ configTabs.advanced ? '▲' : '▼' }}</text>
					</view>
					<view class="config-collapse-body" v-show="configTabs.advanced">
						<view class="config-item"><text>横向左转</text><input v-model="config.crossLeft" /></view>
						<view class="config-item"><text>横向右转</text><input v-model="config.crossRight" /></view>
						<view class="config-item"><text>寻列下方</text><input v-model="config.findCol" /></view>
						<view class="config-item"><text>左转回正(向上)</text><input v-model="config.alignLeft" /></view>
						<view class="config-item"><text>右转回正(向上)</text><input v-model="config.alignRight" /></view>
						<view class="config-item"><text>进方块中心</text><input v-model="config.enterBlock" /></view>
						<view class="config-item"><text>中心停留</text><input v-model="config.pauseBlock" /></view>
						<view class="config-item"><text>穿出到路口</text><input v-model="config.exitBlock" /></view>
						<view class="config-item"><text>直行入终点</text><input v-model="config.goFinish" /></view>
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
				clearCmd: 'K',
				start: 'S',
				go1: '1', go2: '2', go3: '3', go4: '1', go5: '2',
				time1: '1', time2: '2', time3: '3',
				left: '8', right: 'B', stop: 'P', light: 'D',
				crossLeft: '7', crossRight: 'A', findCol: 'S',
				alignLeft: '8', alignRight: 'B', enterBlock: '1',
				pauseBlock: 'P', exitBlock: '1', goFinish: '3'
			},

			configTabs: {
				custom: false, base: true, lines: false, time: false, advanced: false
			},

			customModuleIndex: 0,
			moduleOptions: [
				{ id: 'base', name: '操作指令' }, { id: 'lines', name: '直走到路口' },
				{ id: 'time', name: '直走时间' }, { id: 'advanced', name: '路径规划动作集' },
				{ id: 'routes', name: '我的预设路线' }
			],
			moduleNamesMap: {
				base: '操作指令', lines: '直走到路口', time: '直走时间', advanced: '路径规划动作集', routes: '我的预设路线'
			},
			newCustomName: '', newCustomCode: '',
			customCommands: { base: [], lines: [], time: [], advanced: [], routes: [] },

			planMode: 'auto',
			numberRows: [ [1, 2, 3, 4, 5], [1, 2, 3, 4, 5], [1, 2, 3, 4, 5] ],
			resultCells: [21, 22, 23, 24, 25],
			selectedNumbers: [null, null, null],
			selectedCols: [null, null, null],
			selectedOperators: [null, null],
			selectedResult: 21,
			bestPathObj: null,
			svgPoints: ''
		}
	},
	computed: {
		filteredDeviceList() {
			if (this.filterIndex === 1) return this.deviceList.filter(d => d.protocol === 'BLE')
			if (this.filterIndex === 2) return this.deviceList.filter(d => d.protocol === 'SPP')
			return this.deviceList
		},
		manualResultText() {
			const [a, b, c] = this.selectedNumbers
			const [op1, op2] = this.selectedOperators
			if (a === null || b === null || c === null) return '请先选三层数字'
			if (!op1 || !op2) return '请继续选择两层运算符'
			const calc = this.evaluateTrack2Expression(a, op1, b, op2, c)
			if (!calc.valid) return '公式无效'
			return `${calc.formula} = ${this.formatTrack2Result(calc.result)}`
		}
	},
	onLoad(options) {
		const app = getApp()
		if (!app.globalData) app.globalData = {}

		if (app.globalData.btState && app.globalData.btState.isConnected) {
			this.isConnected = app.globalData.btState.isConnected;
			this.deviceId = app.globalData.btState.deviceId;
			this.deviceName = app.globalData.btState.deviceName;
			this.protocol = app.globalData.btState.protocol;
			this.serviceId = app.globalData.btState.serviceId;
			this.characteristicId = app.globalData.btState.characteristicId;
		}

		this.initBluetooth()
		this.registerSppReceiver()

		if (this.protocol === 'SPP') {
			if (app.globalData && app.globalData.sppSocket) {
				this.isConnected = true
				this.startSPPCheck()
			}
		} else if (this.protocol === 'BLE' && this.isConnected) {
			this.startBLECheck()
		}

		uni.$off('bluetoothDisconnected')
		uni.$on('bluetoothDisconnected', () => {
			this.handleInternalDisconnect('蓝牙通信链路已断开')
		})

		this.bleStateChangeCallback = (res) => {
			if (!res.connected && this.deviceId && res.deviceId.toLowerCase() === this.deviceId.toLowerCase()) {
				this.handleInternalDisconnect('BLE 设备已断开')
			}
		};
		uni.onBLEConnectionStateChange(this.bleStateChangeCallback)

		const localConfig = uni.getStorageSync('bt_char_config_track2')
		if (localConfig && localConfig.clearCmd !== undefined) {
			this.config = Object.assign(this.config, localConfig)
		} else {
			uni.setStorageSync('bt_char_config_track2', this.config)
		}

		const localCustom = uni.getStorageSync('bt_custom_commands_track2')
		if (localCustom) this.customCommands = Object.assign(this.customCommands, localCustom)
		
		const savedQueueState = uni.getStorageSync('bt_queue_state_track2')
		if (savedQueueState) this.commandQueue = savedQueueState
		
		this.refreshTrack2Svg()
	},
	onUnload() {
		uni.$off('bluetoothDisconnected')
		if (this.bleStateChangeCallback) {
			uni.offBLEConnectionStateChange(this.bleStateChangeCallback)
		}
		this.clearAllTimers()
		this.unregisterSppReceiver()
		this.stopScan()
	},
	methods: {
		goToTrack1() {
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
				url: '/pages/index/index',
				fail: () => { uni.showToast({ title: '跳转失败', icon: 'none' }); }
			})
		},

		initBluetooth() {
			uni.openBluetoothAdapter({
				success: () => console.log('蓝牙初始化成功'),
				fail: () => {
					uni.showModal({
						title: '蓝牙异常', content: '请开启蓝牙及位置权限，是否前往设置？',
						success: (res) => {
							if (res.confirm && this.isAndroid) {
								try {
									let main = plus.android.runtimeMainActivity()
									let Intent = plus.android.importClass('android.content.Intent')
									let Settings = plus.android.importClass('android.provider.Settings')
									let intent = new Intent(Settings.ACTION_BLUETOOTH_SETTINGS)
									main.startActivity(intent)
								} catch (e) {}
							}
						}
					})
				}
			})
		},
		toggleScan() {
			this.showDeviceList = true
			if (this.isScanning) {
				this.stopScan()
			} else {
				uni.getBluetoothAdapterState({
					success: (res) => {
						if (res.available) { this.startScan() } else {
							uni.showToast({ title: '蓝牙不可用', icon: 'none' })
							this.initBluetooth()
						}
					},
					fail: () => this.initBluetooth()
				})
			}
		},
		startScan() {
			this.deviceList = []
			this.isScanning = true
			uni.startBluetoothDevicesDiscovery({
				allowDuplicatesKey: false,
				success: () => {
					uni.onBluetoothDeviceFound((res) => {
						res.devices.forEach(device => {
							if (!device.name && !device.localName) return
							const existIndex = this.deviceList.findIndex(d => d.deviceId === device.deviceId)
							if (existIndex > -1) {
								this.$set(this.deviceList[existIndex], 'RSSI', device.RSSI)
							} else {
								device.protocol = 'BLE'
								this.deviceList.unshift(device)
							}
						})
					})
				}
			})
			if (this.isAndroid) setTimeout(() => this.getAndroidPairedDevices(), 300)
			setTimeout(() => this.stopScan(), 15000)
		},
		getAndroidPairedDevices() {
			try {
				let BluetoothAdapter = plus.android.importClass('android.bluetooth.BluetoothAdapter')
				let BAdapter = BluetoothAdapter.getDefaultAdapter()
				if (!BAdapter || !BAdapter.isEnabled()) return
				let pairedDevices = BAdapter.getBondedDevices()
				if (pairedDevices) {
					plus.android.importClass('android.bluetooth.BluetoothDevice')
					let list = plus.android.invoke(pairedDevices, 'toArray')
					if (list && list.length > 0) {
						for (let i = 0; i < list.length; i++) {
							let device = list[i]
							plus.android.importClass(device)
							let mac = device.getAddress()
							let name = device.getName() || '经典蓝牙设备'
							if (!this.deviceList.find(d => d.deviceId === mac)) {
								this.deviceList.push({ name: name + ' (已配对)', deviceId: mac, protocol: 'SPP', RSSI: 'Unknown', isPaired: true })
							}
						}
					}
				}
			} catch (e) {}
		},
		stopScan() {
			if (this.isScanning) { this.isScanning = false; uni.stopBluetoothDevicesDiscovery() }
		},
		connectTo(device) {
			if (this.isConnected) return uni.showToast({ title: '请先断开当前连接', icon: 'none' })
			this.connectingId = device.deviceId
			this.stopScan()

			if (device.protocol === 'SPP') {
				this.connectSPP(device)
			} else {
				uni.createBLEConnection({
					deviceId: device.deviceId,
					timeout: 10000,
					success: () => {
						this.deviceId = device.deviceId
						this.deviceName = device.name || '未知设备'
						this.protocol = 'BLE'
						this.isConnected = true
						this.showDeviceList = false
						this.initBLEService()
						this.startBLECheck()
						uni.showToast({ title: '连接成功' })
					},
					fail: (err) => { uni.showModal({ title: '连接失败', content: '错误码: ' + err.errCode, showCancel: false }) },
					complete: () => { this.connectingId = '' }
				})
			}
		},
		connectSPP(device) {
			setTimeout(() => {
				try {
					let BluetoothAdapter = plus.android.importClass('android.bluetooth.BluetoothAdapter')
					let BAdapter = BluetoothAdapter.getDefaultAdapter()
					let remoteDevice = BAdapter.getRemoteDevice(device.deviceId)
					let UUID = plus.android.importClass('java.util.UUID')
					let sppUuid = UUID.fromString('00001101-0000-1000-8000-00805F9B34FB')
					let socket = remoteDevice.createRfcommSocketToServiceRecord(sppUuid)
					plus.android.importClass(socket)
					socket.connect()

					getApp().globalData.sppSocket = socket
					this.deviceId = device.deviceId
					this.deviceName = device.name || '经典蓝牙串口'
					this.protocol = 'SPP'
					this.isConnected = true
					this.showDeviceList = false
					this.startSPPCheck()
					uni.showToast({ title: 'SPP连接成功' })
				} catch (e) {
					uni.showModal({ title: '连接失败', content: '请确保设备已配对并开启', showCancel: false })
				}
				this.connectingId = ''
			}, 200)
		},
		startSPPCheck() {
			if (this.sppCheckTimer) clearInterval(this.sppCheckTimer)
			this.sppCheckTimer = setInterval(() => {
				if (getApp().globalData.sppSocket && this.isConnected) {
					try {
						let alive = plus.android.invoke(getApp().globalData.sppSocket, 'isConnected')
						if (!alive) throw 'Socket Disconnected'
					} catch (e) { this.handleInternalDisconnect('SPP 蓝牙链路已失效') }
				}
			}, 3000)
		},
		registerSppReceiver() {
			if (!this.isAndroid) return
			try {
				let main = plus.android.runtimeMainActivity()
				let IntentFilter = plus.android.importClass('android.content.IntentFilter')
				let BluetoothDevice = plus.android.importClass('android.bluetooth.BluetoothDevice')

				this.sppReceiver = plus.android.implements('io.dcloud.feature.internal.reflect.BroadcastReceiver', {
					onReceive: (context, intent) => {
						plus.android.importClass(intent)
						let action = intent.getAction()
						if (action == BluetoothDevice.ACTION_ACL_DISCONNECTED) {
							let device = intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE)
							if (device) {
								plus.android.importClass(device)
								let mac = plus.android.invoke(device, 'getAddress')
								if (this.isConnected && mac === this.deviceId) { this.handleInternalDisconnect('设备已物理断开') }
							}
						}
					}
				})
				let filter = new IntentFilter()
				filter.addAction(BluetoothDevice.ACTION_ACL_DISCONNECTED)
				main.registerReceiver(this.sppReceiver, filter)
			} catch (e) {}
		},
		unregisterSppReceiver() {
			if (this.isAndroid && this.sppReceiver) {
				try {
					let main = plus.android.runtimeMainActivity()
					main.unregisterReceiver(this.sppReceiver)
					this.sppReceiver = null
				} catch (e) {}
			}
		},
		startBLECheck() {
			if (this.bleHeartbeatTimer) clearInterval(this.bleHeartbeatTimer)
			this.bleHeartbeatTimer = setInterval(() => {
				if (this.isConnected && this.protocol === 'BLE') {
					uni.getBLEDeviceRSSI({ deviceId: this.deviceId, fail: () => this.handleInternalDisconnect('BLE 连接已丢失') })
				}
			}, 5000)
		},
		handleInternalDisconnect(msg = '连接已断开') {
			this.isConnected = false
			this.deviceName = '未连接设备'

			const app = getApp()
			if (app.globalData) app.globalData.btState = null

			if (this.protocol === 'BLE' && this.deviceId) { uni.closeBLEConnection({ deviceId: this.deviceId }) }

			if (app.globalData && app.globalData.sppSocket) {
				try { app.globalData.sppSocket.close() } catch (e) {}
				app.globalData.sppSocket = null
			}
			this.deviceId = ''; this.protocol = ''; this.clearAllTimers()
			uni.showModal({ title: '提示', content: msg, showCancel: false })
		},
		disconnectDevice() {
			if (!this.isConnected) return
			this.handleInternalDisconnect('已主动断开连接')
		},
		clearAllTimers() {
			if (this.sppCheckTimer) clearInterval(this.sppCheckTimer)
			if (this.bleHeartbeatTimer) clearInterval(this.bleHeartbeatTimer)
		},
		getRssiClass(rssi) {
			if (rssi === 'Unknown') return 'rssi-unknown'
			let val = parseInt(rssi)
			if (isNaN(val)) return 'rssi-unknown'
			if (val >= -60) return 'rssi-good'
			if (val >= -80) return 'rssi-normal'
			return 'rssi-bad'
		},
		filterChange(e) { this.filterIndex = parseInt(e.detail.value) },

		clearMap() {
			this.bestPathObj = null
			this.svgPoints = ''
			this.selectedNumbers = [null, null, null]
			this.selectedCols = [null, null, null]
			this.selectedOperators = [null, null]
			this.selectedResult = this.resultCells[0]
			this.refreshTrack2Svg()
			uni.showToast({ title: '地图已清空', icon: 'none' })
		},
		switchMode(mode) {
			this.planMode = mode
			this.bestPathObj = null
			if (mode === 'auto') this.selectedOperators = [null, null]
			this.refreshTrack2Svg()
		},
		isNumberActive(rowIndex, val) { return this.selectedNumbers[rowIndex] === val },
		isOperatorActive(layerIndex, op) { return this.selectedOperators[layerIndex] === op },
		isResultActive(val) { return this.selectedResult === val },
		onNumberTap(rowIndex, val, colIndex) {
			this.$set(this.selectedNumbers, rowIndex, val)
			this.$set(this.selectedCols, rowIndex, colIndex + 1)
			this.bestPathObj = null
			this.refreshTrack2Svg()
		},
		onOperatorTap(layerIndex, op) {
			if (this.planMode !== 'manual') {
				uni.showToast({ title: '自动模式下运算符会自动匹配', icon: 'none' })
				return
			}
			this.$set(this.selectedOperators, layerIndex, op)
			this.bestPathObj = null
			this.refreshTrack2Svg()
		},
		onResultTap(val) {
			this.selectedResult = val
			this.bestPathObj = null
			this.refreshTrack2Svg()
		},
		calculateTrack2Path() {
			const [a, b, c] = this.selectedNumbers
			const [col1, col2, col3] = this.selectedCols
			if (a === null || b === null || c === null) { return uni.showToast({ title: '请先在三层数字区各选一个数字', icon: 'none' }) }
			if (!this.selectedResult) { return uni.showToast({ title: '请先点选顶部结果区', icon: 'none' }) }

			const ops = ['+', '-', '×', '÷']
			let candidates = []

			ops.forEach(op1 => {
				ops.forEach(op2 => {
					const calc = this.evaluateTrack2Expression(a, op1, b, op2, c)
					if (!calc.valid) return
					const intResult = Math.round(calc.result)
					if (Math.abs(calc.result - intResult) > 0.000001) return
					if (intResult === this.selectedResult) {
						candidates.push({
							op1, op2, result: intResult, formula: calc.formula,
							equation: `${calc.formula} = ${intResult}`,
							distance: this.calcTrack2Distance(col1, op1, col2, op2, col3, this.getResultCol(this.selectedResult))
						})
					}
				})
			})

			if (!candidates.length) {
				this.bestPathObj = null
				this.selectedOperators = [null, null]
				this.refreshTrack2Svg()
				uni.showToast({ title: '当前三层数字无法算出这个结果', icon: 'none' })
				return
			}

			candidates.sort((a, b) => a.distance - b.distance)
			const best = candidates[0]
			this.selectedOperators = [best.op1, best.op2]
			this.bestPathObj = best
			this.generateTrack2Queue()
			this.refreshTrack2Svg()
			uni.showToast({ title: '已生成最优公式与路径', icon: 'success' })
		},
		generateManualPathCmd() {
			const [a, b, c] = this.selectedNumbers
			const [op1, op2] = this.selectedOperators
			if (a === null || b === null || c === null) return uni.showToast({ title: '请先选三层数字', icon: 'none' })
			if (!op1 || !op2) return uni.showToast({ title: '请先选两层运算符', icon: 'none' })

			const calc = this.evaluateTrack2Expression(a, op1, b, op2, c)
			if (!calc.valid) return uni.showToast({ title: '当前公式无效', icon: 'none' })
			const intResult = Math.round(calc.result)
			if (Math.abs(calc.result - intResult) > 0.000001) return uni.showToast({ title: '当前公式结果不是整数，不能进结果区', icon: 'none' })
			if (intResult !== this.selectedResult) return uni.showToast({ title: '公式结果和顶部结果区不一致', icon: 'none' })

			this.bestPathObj = {
				op1, op2, result: intResult, formula: calc.formula, equation: `${calc.formula} = ${intResult}`,
				distance: this.calcTrack2Distance(this.selectedCols[0], op1, this.selectedCols[1], op2, this.selectedCols[2], this.getResultCol(this.selectedResult))
			}
			this.generateTrack2Queue()
			this.refreshTrack2Svg()
			uni.showToast({ title: '手动路径已生成', icon: 'success' })
		},
		evaluateTrack2Expression(a, op1, b, op2, c) {
			const priority = (op) => { return (op === '×' || op === '÷') ? 2 : 1 }
			const calc = (x, op, y) => {
				if (op === '+') return { valid: true, value: x + y }
				if (op === '-') return { valid: true, value: x - y }
				if (op === '×') return { valid: true, value: x * y }
				if (op === '÷') { return y === 0 ? { valid: false, value: null } : { valid: true, value: x / y } }
				return { valid: false, value: null }
			}
			try {
				if (priority(op1) >= priority(op2)) {
					const step1 = calc(a, op1, b)
					if (!step1.valid) return { valid: false, result: null, formula: '' }
					const step2 = calc(step1.value, op2, c)
					if (!step2.valid) return { valid: false, result: null, formula: '' }
					return { valid: true, result: step2.value, formula: `(${a} ${op1} ${b}) ${op2} ${c}` }
				} else {
					const step1 = calc(b, op2, c)
					if (!step1.valid) return { valid: false, result: null, formula: '' }
					const step2 = calc(a, op1, step1.value)
					if (!step2.valid) return { valid: false, result: null, formula: '' }
					return { valid: true, result: step2.value, formula: `${a} ${op1} (${b} ${op2} ${c})` }
				}
			} catch (e) { return { valid: false, result: null, formula: '' } }
		},
		formatTrack2Result(val) {
			if (val === null || val === undefined) return ''
			if (Math.abs(val - Math.round(val)) < 0.000001) return String(Math.round(val))
			return Number(val).toFixed(2)
		},
		getOpCol(op) {
			if (op === '+') return 1; if (op === '-') return 2; if (op === '×') return 4; if (op === '÷') return 5; return 3;
		},
		getResultCol(result) {
			const idx = this.resultCells.indexOf(result); return idx === -1 ? 3 : idx + 1;
		},
		calcTrack2Distance(col1, op1, col2, op2, col3, resultCol) {
			const opCol1 = this.getOpCol(op1)
			const opCol2 = this.getOpCol(op2)
			return Math.abs(col3 - opCol2) + Math.abs(opCol2 - col2) + Math.abs(col2 - opCol1) + Math.abs(opCol1 - col1) + Math.abs(col1 - resultCol)
		},
		appendMoveToColumn(queue, currentCol, targetCol, label) {
			if (!targetCol || currentCol === targetCol) return targetCol || currentCol
			if (targetCol > currentCol) {
				queue.push({ name: `${label}：右转横移`, code: this.config.crossRight })
				queue.push({ name: `${label}：寻至目标列`, code: this.config.findCol })
				queue.push({ name: `${label}：左转回正`, code: this.config.alignLeft })
			} else {
				queue.push({ name: `${label}：左转横移`, code: this.config.crossLeft })
				queue.push({ name: `${label}：寻至目标列`, code: this.config.findCol })
				queue.push({ name: `${label}：右转回正`, code: this.config.alignRight })
			}
			return targetCol
		},
		appendBlockAction(queue, name) {
			queue.push({ name: `${name}：进入方块`, code: this.config.enterBlock })
			queue.push({ name: `${name}：中心停留`, code: this.config.pauseBlock })
			queue.push({ name: `${name}：穿出到上方路口`, code: this.config.exitBlock })
		},
		appendOperatorAction(queue, name) {
			queue.push({ name: `${name}：进入符号通道`, code: this.config.enterBlock })
			queue.push({ name: `${name}：完成符号选择`, code: this.config.pauseBlock })
			queue.push({ name: `${name}：穿出到上方路口`, code: this.config.exitBlock })
		},
		generateTrack2Queue() {
			const [a, b, c] = this.selectedNumbers
			const [op1, op2] = this.selectedOperators
			const [col1, col2, col3] = this.selectedCols
			let queue = []
			let currentCol = 3

			queue.push({ name: '启动运行', code: this.config.start })
			currentCol = this.appendMoveToColumn(queue, currentCol, col3, `前往底层数字${c}`)
			this.appendBlockAction(queue, `底层数字 ${c}`)
			currentCol = this.appendMoveToColumn(queue, currentCol, this.getOpCol(op2), `前往第二层符号 ${op2}`)
			this.appendOperatorAction(queue, `第二层符号 ${op2}`)
			currentCol = this.appendMoveToColumn(queue, currentCol, col2, `前往中层数字${b}`)
			this.appendBlockAction(queue, `中层数字 ${b}`)
			currentCol = this.appendMoveToColumn(queue, currentCol, this.getOpCol(op1), `前往第一层符号 ${op1}`)
			this.appendOperatorAction(queue, `第一层符号 ${op1}`)
			currentCol = this.appendMoveToColumn(queue, currentCol, col1, `前往顶层数字${a}`)
			this.appendBlockAction(queue, `顶层数字 ${a}`)
			currentCol = this.appendMoveToColumn(queue, currentCol, this.getResultCol(this.selectedResult), `前往结果区 ${this.selectedResult}`)
			queue.push({ name: `进入结果区 ${this.selectedResult}`, code: this.config.goFinish })
			queue.push({ name: '驶入终点区', code: this.config.goFinish })

			this.commandQueue = queue
			this.saveQueueState()
		},
		
		getColX(col) { const map = { 1: 17.7, 2: 33.9, 3: 50, 4: 66.1, 5: 82.3 }; return map[col] || 50 },
		getOpX(op) { const map = { '+': 17.7, '-': 33.9, '×': 66.1, '÷': 82.3 }; return map[op] || 50 },
		getResultX(result) { const idx = this.resultCells.indexOf(result); const xs = [17.7, 33.9, 50, 66.1, 82.3]; return idx === -1 ? 50 : xs[idx] },
		
		refreshTrack2Svg() {
			let pts = []
			pts.push('50,96')
			if (this.selectedCols[2]) pts.push(`${this.getColX(this.selectedCols[2])},84`)
			if (this.selectedOperators[1]) pts.push(`${this.getOpX(this.selectedOperators[1])},66`)
			if (this.selectedCols[1]) pts.push(`${this.getColX(this.selectedCols[1])},53`)
			if (this.selectedOperators[0]) pts.push(`${this.getOpX(this.selectedOperators[0])},35`)
			if (this.selectedCols[0]) pts.push(`${this.getColX(this.selectedCols[0])},22`)
			if (this.selectedResult) pts.push(`${this.getResultX(this.selectedResult)},8.5`)
			this.svgPoints = pts.join(' ')
		},
		
		saveQueueState() { uni.setStorageSync('bt_queue_state_track2', this.commandQueue) },
		clearQueue() { this.commandQueue = []; this.saveQueueState() },
		
		saveQueueAsMacro() {
			if (this.commandQueue.length === 0) return uni.showToast({ title: '队列为空', icon: 'none' })
			const codes = this.commandQueue.map(i => i.code).join('')
			const defaultName = `算术路线_${new Date().getSeconds()}`
			uni.showModal({
				title: '保存为预设路线', content: defaultName, editable: true, placeholderText: '给这条路线起个名字',
				success: (res) => {
					if (res.confirm) {
						const finalName = res.content || defaultName
						if (!this.customCommands.routes) this.$set(this.customCommands, 'routes', [])
						this.customCommands.routes.push({ name: finalName, code: codes })
						this.saveCustomCommands()
						uni.showToast({ title: '已保存至[我的预设路线]', icon: 'success' })
					}
				}
			})
		},
		onCustomModuleChange(e) { this.customModuleIndex = parseInt(e.detail.value) },
		addCustomCommandToStorage() {
			if (!this.newCustomName || !this.newCustomCode) return uni.showToast({ title: '请填写名称和代码', icon: 'none' })
			const modId = this.moduleOptions[this.customModuleIndex].id
			this.customCommands[modId].push({ name: this.newCustomName, code: this.newCustomCode })
			this.newCustomName = ''; this.newCustomCode = ''; this.saveCustomCommands()
			uni.showToast({ title: '添加成功，已渲染至面板', icon: 'success' })
		},
		removeCustomCommand(modKey, idx) { this.customCommands[modKey].splice(idx, 1); this.saveCustomCommands() },
		saveCustomCommands() { uni.setStorageSync('bt_custom_commands_track2', this.customCommands) },
		toggleConfigTab(tabName) { this.configTabs[tabName] = !this.configTabs[tabName] },
		
		sendSPPData(text, isSilent = false) {
			const app = getApp()
			if (!app.globalData.sppSocket) return this.handleInternalDisconnect('连接已失效')
			try {
				let outStream = app.globalData.sppSocket.getOutputStream()
				plus.android.importClass(outStream)
				let bytes = plus.android.invoke(text, 'getBytes', 'gbk')
				outStream.write(bytes)
				outStream.flush()
				if (!isSilent) {
					uni.showToast({ title: '已发送' })
					this.syncToChatCache('send', text.replace(/\r\n/g, ''))
				}
			} catch (e) { this.handleInternalDisconnect('设备已掉线，发送失败') }
		},
		sendBLEData(text, isSilent = false) {
			if (!this.serviceId || !this.characteristicId) { 
				if(!isSilent) uni.showToast({ title: '初始化特征值...', icon: 'none' }); 
				this.initBLEService(); return 
			}
			const buffer = new ArrayBuffer(text.length)
			const dataView = new DataView(buffer)
			for (let i = 0; i < text.length; i++) dataView.setUint8(i, text.charCodeAt(i))
			uni.writeBLECharacteristicValue({
				deviceId: this.deviceId, serviceId: this.serviceId, characteristicId: this.characteristicId, value: buffer,
				success: () => { 
					if(!isSilent) {
						uni.showToast({ title: '已发送' }); 
						this.syncToChatCache('send', text.replace(/\r\n/g, '')) 
					}
				},
				fail: (err) => { 
					if (err.errCode === 10006) this.handleInternalDisconnect('BLE 连接已断开'); 
					else if(!isSilent) uni.showToast({ title: '发送失败:' + err.errCode, icon: 'none' }) 
				}
			})
		},
		async sendBatch() {
			if (this.commandQueue.length === 0) return
			if (!this.isConnected) return uni.showToast({ title: '蓝牙未连接', icon: 'none' })
			
			const clearCode = this.config.clearCmd || 'K';
			const codes = this.commandQueue.map(i => i.code).join('')
			const dataString = codes + '\r\n'
			
			if (this.protocol === 'SPP') {
				if(clearCode) this.sendSPPData(clearCode, true);
				setTimeout(() => { this.sendSPPData(dataString); }, 200);
			} else {
				if(clearCode) this.sendBLEData(clearCode, true);
				setTimeout(() => { this.sendBLEData(dataString); }, 200);
			}
		},
		syncToChatCache(type, content) {
			const cacheKey = 'chat_history_' + this.deviceId
			let history = uni.getStorageSync(cacheKey) || []
			const now = new Date()
			const timeStr = `${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}:${now.getSeconds().toString().padStart(2, '0')}`
			history.push({ type: type, text: '控制指令: ' + content, time: timeStr })
			uni.setStorageSync(cacheKey, history)
			uni.$emit('refreshChat')
		},
		initBLEService() {
			uni.getBLEDeviceServices({
				deviceId: this.deviceId,
				success: (res) => {
					const mainService = res.services.find(s => s.uuid.includes('FFE0') || s.uuid.includes('FFF0')) || res.services[0]
					this.serviceId = mainService.uuid
					uni.getBLEDeviceCharacteristics({
						deviceId: this.deviceId, serviceId: this.serviceId,
						success: (cRes) => {
							const char = cRes.characteristics.find(c => c.properties.write)
							if (char) this.characteristicId = char.uuid
						}
					})
				}
			})
		},
		addCommand(keyOrCode, name, isCustom = false) {
			let code = isCustom ? keyOrCode : this.config[keyOrCode]; this.commandQueue.push({ name, code }); this.saveQueueState()
		},
		deleteLast() { this.commandQueue.pop(); this.saveQueueState() },
		
		saveConfig() { uni.setStorageSync('bt_char_config_track2', this.config); this.showConfig = false; uni.showToast({ title: '配置已保存' }) },
		
		goToChat() {
			if (!this.isConnected) { uni.showToast({ title: '未连接蓝牙，当前为离线预览模式', icon: 'none' }) }
			uni.navigateTo({ url: `/pages/dueihua/dueihua?deviceName=${encodeURIComponent(this.deviceName)}&protocol=${this.protocol}&deviceId=${this.deviceId}` })
		}
	}
}
</script>

<style>
/* === 通用基础样式无需改动 === */
.container { background-color: #f5f7f9; min-height: 100vh; }
.status-bar { height: var(--status-bar-height); background-color: #ffffff; }
.safe-bottom { height: 40rpx; }
.track-switcher { display: flex; margin: 20rpx 24rpx; background: #e2e8f0; border-radius: 16rpx; padding: 6rpx; }
.track-btn { flex: 1; text-align: center; padding: 18rpx 0; font-size: 28rpx; font-weight: bold; color: #64748b; border-radius: 12rpx; transition: 0.3s; }
.track-btn.active { background: #ffffff; color: #3b82f6; box-shadow: 0 4rpx 10rpx rgba(0, 0, 0, 0.06); }
.connection-header { display: flex; justify-content: space-between; align-items: center; background-color: #ffffff; padding: 24rpx 30rpx; border-bottom: 1rpx solid #eeeeee; position: sticky; top: 0; z-index: 100; }
.btn-config-entry { margin: 0; font-size: 22rpx; background: #f1f5f9; color: #475569; height: 54rpx; line-height: 54rpx; padding: 0 20rpx; }
.conn-status { display: flex; align-items: center; background-color: #f0f9ff; padding: 12rpx 24rpx; border-radius: 40rpx; width: fit-content; }
.status-dot { width: 16rpx; height: 16rpx; border-radius: 50%; background-color: #94a3b8; margin-right: 16rpx; }
.status-dot.active { background-color: #22c55e; box-shadow: 0 0 10rpx rgba(34, 197, 94, 0.4); }
.conn-text { font-size: 26rpx; color: #0369a1; font-weight: 600; }
.toggle-hint { font-size: 20rpx; color: #94a3b8; margin-left: 10rpx; font-weight: normal; }
.scan-panel { background: #ffffff; border-bottom: 1rpx solid #eeeeee; box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.02); }
.scan-controls { display: flex; align-items: center; justify-content: space-between; padding: 20rpx 30rpx; }
.btn-scan { background: linear-gradient(135deg, #007AFF, #0055FF); color: #fff; font-size: 26rpx; border-radius: 50rpx; margin: 0; padding: 0 40rpx; height: 60rpx; line-height: 60rpx; flex-shrink: 0; }
.filter-tag { font-size: 24rpx; color: #555; background: #f1f5f9; padding: 10rpx 24rpx; border-radius: 30rpx; border: 1rpx solid #e2e8f0; }
.btn-disconnect-sm { background: #fee2e2; color: #ef4444; font-size: 22rpx; border-radius: 30rpx; padding: 0 24rpx; height: 50rpx; line-height: 50rpx; margin: 0; font-weight: bold; }
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
.btn-connect { font-size: 24rpx; color: #ffffff; background: #3b82f6; border-radius: 30rpx; margin: 0; padding: 0 30rpx; height: 56rpx; line-height: 56rpx; font-weight: bold; }
.btn-connect[disabled] { background: #f1f5f9 !important; color: #94a3b8 !important; }
.card { background: #ffffff; margin: 24rpx; padding: 32rpx; border-radius: 28rpx; box-shadow: 0 6rpx 24rpx rgba(0, 0, 0, 0.04); }
.card-header { display: flex; align-items: center; margin-bottom: 24rpx; }
.card-header-between { display: flex; justify-content: space-between; align-items: center; margin-bottom: 24rpx; }
.card-title { font-size: 34rpx; font-weight: 800; color: #1e293b; }
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
.section-title { font-size: 28rpx; font-weight: bold; color: #334155; margin-bottom: 20rpx; display: flex; align-items: center; }
.title-arrow, .title-icon { margin-right: 10rpx; font-size: 32rpx; }
.button-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16rpx; }
.button-grid-3 { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16rpx; }
.button-grid-2 { display: grid; grid-template-columns: repeat(2, 1fr); gap: 16rpx; }
.code-hint { font-size: 20rpx; opacity: 0.6; margin-left: 8rpx; font-weight: normal; }
.chat-preview-box { display: flex; justify-content: space-between; align-items: center; background-color: #f0fdf4; border: 2rpx dashed #22c55e; padding: 20rpx 30rpx; border-radius: 20rpx; margin-bottom: 30rpx; }
.chat-content { display: flex; align-items: center; }
.chat-icon { font-size: 40rpx; margin-right: 20rpx; }
.chat-text-area { display: flex; flex-direction: column; }
.chat-label { font-size: 28rpx; font-weight: bold; color: #166534; }
.chat-hint { font-size: 22rpx; color: #15803d; margin-top: 4rpx; }
.arrow-right { font-size: 34rpx; color: #15803d; font-weight: bold; }
.path-planning-box { padding: 10rpx 0; }
.input-group { display: flex; justify-content: space-between; align-items: center; background: #f8fafc; padding: 20rpx; border-radius: 12rpx; margin-bottom: 16rpx; }
.input-label { font-size: 28rpx; color: #475569; font-weight: 500; }
.picker-display { font-size: 28rpx; color: #3b82f6; font-weight: bold; text-align: right; max-width: 420rpx; }
.manual-hint { font-size: 24rpx; color: #f59e0b; text-align: center; margin-top: 10rpx; margin-bottom: 20rpx; font-weight: bold; line-height: 1.7; }
.btn-generate-path { background: #e0e7ff; color: #4338ca; height: 88rpx; line-height: 88rpx; border-radius: 16rpx; font-size: 28rpx; font-weight: bold; margin-top: 20rpx; border: 1px solid #c7d2fe; }
.btn-generate-path:active { background: #c7d2fe; }
.manual-btn { background: #fef3c7 !important; color: #d97706 !important; border-color: #fde68a !important; margin-top: 16rpx; }
.manual-btn:active { background: #fde68a !important; }
.path-result { margin-top: 24rpx; background: #f0fdf4; border-radius: 12rpx; padding: 20rpx; border: 1px dashed #86efac; }
.result-row { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12rpx; }
.result-row:last-child { margin-bottom: 0; }
.result-tag { font-size: 24rpx; color: #166534; }
.result-val { font-size: 28rpx; font-weight: bold; color: #15803d; max-width: 430rpx; text-align: right; word-break: break-all; }
.queue-display-area { min-height: 160rpx; background: #f8fafc; border-radius: 16rpx; padding: 20rpx; margin-bottom: 20rpx; }
.queue-list { display: flex; flex-wrap: wrap; gap: 10rpx; }
.queue-item { background: #3b82f6; color: #fff; padding: 6rpx 16rpx; border-radius: 8rpx; font-size: 24rpx; }
.q-name { margin-right: 8rpx; }
.empty-placeholder { padding: 40rpx 0; text-align: center; }
.empty-subtext { font-size: 24rpx; color: #cbd5e1; }
.mode-tabs { display: flex; background: #f1f5f9; border-radius: 12rpx; padding: 8rpx; margin-bottom: 24rpx; }
.mode-tab { flex: 1; text-align: center; font-size: 28rpx; color: #64748b; padding: 16rpx 0; border-radius: 8rpx; transition: all 0.3s; font-weight: bold; }
.mode-tab.active { background: #ffffff; color: #3b82f6; box-shadow: 0 4rpx 10rpx rgba(0, 0, 0, 0.06); }

/* ================================================= */
/* ====== 核心修正：针对手机端的防裁剪高度与相对定位封装 ====== */
/* ================================================= */

.track2-photo-map {
	position: relative;
	background: #ffffff;
	border: 2px dashed #222; 
	border-radius: 0; 
	overflow: hidden;
}

.track2-svg {
	position: absolute;
	inset: 0;
	width: 100%;
	height: 100%;
	z-index: 20; 
	pointer-events: none;
}

.track2-row {
	display: grid;
	grid-template-columns: 0.6fr 1fr 1fr 1fr 1fr 1fr 0.6fr;
	position: relative;
	z-index: 10;
}

/* ====== 顶部结果区 ====== */
.track2-top-row {
	height: 120rpx; /* 略微加高，适应手机字体 */
	border-bottom: 2px solid #222;
}
.track2-side-finish {
	display: flex;
	align-items: center;
	justify-content: center;
	background: #cce8fb; 
	font-size: 52rpx;
	font-weight: 800;
	color: #111; 
}
.track2-side-finish.left { border-right: 2px solid #222; }
.track2-side-finish.right { border-left: none; }
.track2-result-cell {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	background: #cce8fb; 
	border-right: 2px solid #222; 
	box-sizing: border-box;
}
.track2-result-num { font-size: 54rpx; font-weight: 800; color: #e60012; line-height: 1.1; }
.track2-result-mark { font-size: 38rpx; font-weight: 800; color: #e60012; line-height: 1; margin-top: -2rpx; letter-spacing: 2rpx; }
.track2-result-cell.active { box-shadow: inset 0 0 0 3px rgba(230, 0, 18, 0.7); }

/* ====== 数字行与占位列 ====== */
.track2-number-row {
	height: 120rpx; /* 增加高度防止数字被切 */
	border-bottom: 2px solid #222;
}
.track2-side-blank {
	background: #ffffff;
	border-right: 2px solid #222; 
	z-index: 5;
}
.track2-side-blank:last-child {
	border-right: none;
	border-left: none; 
}
.track2-number-cell {
	display: flex;
	align-items: center;
	justify-content: center;
	background: #ffffff;
	border-right: 2px solid #222; 
	box-sizing: border-box;
}
.track2-number-cell text { font-size: 56rpx; font-weight: 800; color: #cccccc; line-height: 1; }
.track2-number-cell.active { background: #fff1f1; box-shadow: inset 0 0 0 3px rgba(230, 0, 18, 0.5); }
.track2-number-cell.active text { color: #e60012; }

/* ====== 关键重构：将 SVG 从 Grid 中解脱出来的包装层 ====== */
.track2-op-wrapper {
	position: relative;
	height: 220rpx; /* 从180加到220，给三个行号充足的呼吸空间 */
	background: #ffffff;
	border-bottom: 2px solid #222;
}

.h-full {
	height: 100%;
	border-bottom: none !important;
}

.track2-op-wrapper .track2-side-blank {
	border: none !important;
	background: transparent !important;
}

/* 绘制贝塞尔曲线边界与背景 */
.lane-path {
	stroke: #222;
	stroke-width: 2px; 
	stroke-linejoin: round;
	vector-effect: non-scaling-stroke;
	transition: fill 0.2s ease-in-out;
}

.lane-path.plus { fill: #e60012; }
.lane-path.minus { fill: #fff100; }
.lane-path.multiply { fill: #009944; }
.lane-path.divide { fill: #00a0e9; }

.lane-path.plus.active { fill: #ff6b6b; }
.lane-path.minus.active { fill: #ffff80; }
.lane-path.multiply.active { fill: #4ade80; }
.lane-path.divide.active { fill: #38bdf8; }

/* ====== 符号布局与中心曲线对齐 ====== */
.track2-op-empty-zone {
	z-index: 5; 
}

.track2-op-text-zone {
	position: relative;
	z-index: 5;
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: space-around;
	cursor: pointer;
	height: 100%;
}

.track2-op-text-zone text {
	font-size: 50rpx; /* 适当减小防止溢出 */
	font-weight: 800;
	color: #ffffff;
	line-height: 1.2;
}

.track2-op-text-zone.minus text {
	color: #d4d4d4;
}

.top-op { transform: translateX(-24rpx); }
.mid-op { transform: translateX(0); }
.bot-op { transform: translateX(24rpx); }

/* ====== 底部启动区 ====== */
.track2-start-row { height: 120rpx; }
.track2-start-cell { display: flex; align-items: center; justify-content: center; background: #009944; border-right: 2px solid #222; box-sizing: border-box; }
.track2-start-cell:last-child { border-right: none; }
.track2-start-cell.text-cell { font-size: 60rpx; font-weight: 800; color: #111; }

/* === 总体配置样式保持不变 === */
.config-mask { position: fixed; top: 0; left: 0; right: 0; bottom: 0; background: rgba(0, 0, 0, 0.6); z-index: 9999; display: flex; align-items: center; justify-content: center; }
.config-panel { width: 85%; background: #ffffff; border-radius: 30rpx; padding: 40rpx; height: 800rpx; display: flex; flex-direction: column; }
.config-title { font-weight: bold; margin-bottom: 10rpx; text-align: center; color: #1e293b; font-size: 30rpx; }
.config-scroll { flex: 1; height: 1px; overflow-y: scroll; padding: 10rpx 0; }
.config-collapse-header { display: flex; justify-content: space-between; align-items: center; background: #f1f5f9; padding: 20rpx 30rpx; border-radius: 12rpx; margin-top: 16rpx; font-size: 28rpx; color: #334155; font-weight: bold; }
.config-collapse-body { background: #ffffff; padding: 10rpx 20rpx; border: 1px solid #f1f5f9; border-top: none; border-radius: 0 0 12rpx 12rpx; }
.config-item { display: flex; align-items: center; justify-content: space-between; margin-bottom: 20rpx; border-bottom: 1rpx dashed #f1f5f9; padding-bottom: 12rpx; padding-top: 10rpx; }
.config-item text { font-size: 26rpx; color: #64748b; }
.config-item input { width: 160rpx; background: #f8fafc; text-align: center; font-size: 26rpx; border-radius: 10rpx; padding: 10rpx; border: 1rpx solid #e2e8f0; }
.custom-cmd-form { padding: 14rpx 0; }
.custom-picker { background: #f8fafc; border: 1rpx solid #e2e8f0; border-radius: 12rpx; padding: 18rpx 20rpx; font-size: 24rpx; color: #475569; margin-bottom: 16rpx; }
.custom-input-row { display: flex; gap: 16rpx; margin-bottom: 16rpx; }
.c-input { flex: 1; background: #f8fafc; border: 1rpx solid #e2e8f0; border-radius: 12rpx; padding: 16rpx 18rpx; font-size: 24rpx; }
.btn-add-custom { background: #e0f2fe; color: #0284c7; height: 74rpx; line-height: 74rpx; font-size: 26rpx; font-weight: bold; }
.custom-list { margin-top: 16rpx; }
.custom-list-item { background: #f8fafc; border-radius: 12rpx; padding: 16rpx; margin-bottom: 14rpx; }
.custom-mod-title { font-size: 24rpx; color: #334155; font-weight: bold; margin-bottom: 10rpx; }
.custom-cmd-row { display: flex; justify-content: space-between; align-items: center; padding: 12rpx 0; border-bottom: 1rpx dashed #e2e8f0; }
.custom-cmd-row:last-child { border-bottom: none; }
.cc-name { font-size: 24rpx; color: #475569; }
.cc-del { font-size: 22rpx; color: #ef4444; }
.config-footer { display: flex; gap: 20rpx; margin-top: 20rpx; }
.btn-exit-config, .btn-save-config { flex: 1; height: 84rpx; line-height: 84rpx; font-size: 28rpx; font-weight: bold; }
.btn-exit-config { background: #e5e7eb; color: #475569; }
.btn-save-config { background: linear-gradient(to right, #10b981, #3b82f6); color: #fff; }
</style>