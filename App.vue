<template>
	<div class="lottery-container">
		<!-- 背景音乐控制区域 -->
		<div class="music-control">
			<div class="music-btn-group">
				<button class="music-btn" @click="toggleBackgroundMusic" :title="isMusicPlaying ? '暂停音乐' : '播放音乐'">
					{{ isMusicPlaying ? '🔊' : '🔇' }}
				</button>
				<button class="volume-toggle-btn" @click="toggleVolumeControl" :title="showVolumeControl ? '隐藏音量控制' : '显示音量控制'">
					🎚️
				</button>
			</div>
			<div v-if="showVolumeControl" class="volume-control">
				<span class="volume-icon">🔉</span>
				<input 
					type="range" 
					min="0" 
					max="100" 
					:value="musicVolume"
					@input="adjustMusicVolume"
					class="volume-slider"
					title="调节音量"
				/>
				<span class="volume-value">{{ musicVolume }}%</span>
			</div>
		</div>

		<!-- 标题区域 -->
		<div class="header">
			<div class="firework"></div>
			<h1 class="title">
				<span class="title-icon">🎊</span>
				元旦晚会幸运大抽奖
				<span class="title-icon">🎉</span>
			</h1>
			<p class="subtitle">2026 · 新年快乐</p>
			<p class="tip">按 Enter 键开始抽奖</p>
		</div>

		<!-- 抽奖卡片区域 -->
		<div class="prize-grid">
			<div v-for="(prize, index) in prizes" :key="index" class="prize-card"
				:class="{ 'highlight': currentIndex === index, 'winner': winnerIndex === index }">
				<div class="prize-number">{{ index + 1 }}</div>
				<div class="prize-title">{{ prize.title }}</div>
				<div class="prize-desc">{{ prize.desc }}</div>
			</div>
		</div>

		<!-- 抽奖按钮 -->
		<button class="lottery-btn" @click="startLottery" :disabled="isRunning">
			{{ isRunning ? '抽奖中...' : '开始抽奖' }}
		</button>

		<!-- 中奖弹窗 -->
		<div v-if="showModal" class="modal-overlay" @click="closeModal">
			<div class="modal-content" @click.stop>
				<div class="modal-header">
					<div class="congratulation">恭喜中奖</div>
					<div class="trophy">🏆</div>
				</div>
				<div class="modal-body">
					<div class="winner-title">{{ prizes[winnerIndex].title }}</div>
					<div class="winner-desc">{{ prizes[winnerIndex].desc }}</div>
				</div>
				<button class="close-btn" @click="closeModal">确定</button>
			</div>
		</div>
	</div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

// 奖品数据接口定义
interface Prize {
	title: string
	desc: string
}

// ==================== 音效配置区域 ====================
// 可以设置为：1. 在线音效链接  2. base64编码  3. 本地文件路径
// 格式示例：
// - 在线链接: 'https://example.com/sound.mp3'
// - base64: 'data:audio/mp3;base64,//uQx...'
// - 本地文件: '/sounds/lottery.mp3'

const LOTTERY_SOUND_URL = 'public/lottery.mov' // 抽奖音效地址（留空则使用默认音效）
const WIN_SOUND_URL = 'public/win.mp3'     // 中奖音效地址（胜利欢呼声）
const BACKGROUND_SOUND_URL = 'public/background.m4a' // 背景音乐地址（留空则使用默认音效）

// =====================================================

// 音效对象
let lotteryAudio: HTMLAudioElement | null = null
let winAudio: HTMLAudioElement | null = null
let backgroundAudio: HTMLAudioElement | null = null

// Web Audio API 备用（默认音效）
let audioContext: AudioContext | null = null

// 初始化音频
const initAudio = () => {
	// 抽奖音效（每次选中播放一次，不循环）
	if (LOTTERY_SOUND_URL) {
		lotteryAudio = new Audio(LOTTERY_SOUND_URL)
		lotteryAudio.volume = 0.4
	}
	
	// 中奖音效
	if (WIN_SOUND_URL) {
		winAudio = new Audio(WIN_SOUND_URL)
		winAudio.volume = 0.7
	}
	
	// 背景音乐
	if (BACKGROUND_SOUND_URL) {
		backgroundAudio = new Audio(BACKGROUND_SOUND_URL)
		backgroundAudio.loop = true
		backgroundAudio.volume = musicVolume.value / 100 // 使用设置的音量
	}
}

// 初始化音频上下文（用于默认音效）
const initAudioContext = () => {
	if (!audioContext) {
		audioContext = new (window.AudioContext || (window as any).webkitAudioContext)()
	}
	return audioContext
}

// 奖品数据数组（16个奖品）
const prizes = ref<Prize[]>([
	{ title: '元旦特特特等奖', desc: '两位老师准备惊喜一份' },
	{ title: '元旦特特等奖', desc: '任选一位师兄师姐约会一次' },
	{ title: '元旦特等奖', desc: '旺旺大礼包' },
	{ title: '求真务实', desc: '获得6.6元红包' },
	{ title: '粮食补给包', desc: '解锁食堂限定石锅饭1份' },
	{ title: '没事哒', desc: '获得溜溜梅一份' },
	{ title: '音乐自由', desc: '获得一个月QQ音乐会员' },
	{ title: '专属应援', desc: '主席团成员为你合唱《恭喜发财》' },
	{ title: '路漫漫其修远兮', desc: '再获得一次抽奖机会' },
	{ title: '吾将上下而求索', desc: '邀请一位朋友来抽一次奖' },
	{ title: '马上下班', desc: '现在离开晚会现场' },
	{ title: '表情小子', desc: '现场拍摄3张搞怪表情发至研会群' },
	{ title: '显眼包', desc: '现场即兴唱一首歌' },
	{ title: '干脆面不嘻嘻', desc: '干脆面一包' },
	{ title: '干脆面嘻嘻', desc: '干脆面一箱' },
	{ title: '多喝热水', desc: '保温杯一个' }
])

// 抽奖状态
const isRunning = ref(false)
const currentIndex = ref(-1)
const winnerIndex = ref(-1)
const showModal = ref(false)

// 背景音乐状态
const isMusicPlaying = ref(false)
const musicVolume = ref(30) // 背景音乐音量（0-100）
const showVolumeControl = ref(false) // 是否显示音量控制面板

// 抽奖速度配置（模式：线性减速，最后三次单独控制速度）
const LOTTERY_INITIAL_SPEED = 100 // 抽奖的初始速度（毫秒），
const LOTTERY_FINAL_SPEED = 400 // 第-3次抽奖的最终速度（毫秒）
const LOTTERY_TOTAL_JUMPS = 20 // 总跳转次数
const LOTTERY_LAST_THREE_SPEED = 750 // 最后三次的速度（毫秒），单独控制

// 开始抽奖
const startLottery = () => {
	if (isRunning.value) return

	isRunning.value = true
	currentIndex.value = Math.floor(Math.random() * 16) // 随机起始位置
	winnerIndex.value = -1
	showModal.value = false

	// 计算最终中奖索引（均等概率）
	const finalIndex = Math.floor(Math.random() * 16)

	let currentStep = 0

	// 执行下一次跳转
	const nextJump = () => {
		currentStep++
		
		// 随机选择下一个卡片（排除当前卡片）
		let nextIndex = currentIndex.value
		while (nextIndex === currentIndex.value) {
			nextIndex = Math.floor(Math.random() * 16)
		}
		currentIndex.value = nextIndex
		
		// 每次选中卡片时播放抽奖音效
		playLotterySound()

		// 到达最后一步，显示最终结果
		if (currentStep >= LOTTERY_TOTAL_JUMPS) {
			currentIndex.value = finalIndex
			winnerIndex.value = finalIndex	

			// 延迟500ms显示弹窗
			setTimeout(() => {
				showModal.value = true
				isRunning.value = false
				// 播放中奖音效
				playWinSound()
			}, 500)
			return
		}

		// 计算当前速度
		let currentSpeed: number
		
		// 最后三次使用单独的速度
		if (currentStep >= LOTTERY_TOTAL_JUMPS - 3) {
			currentSpeed = LOTTERY_LAST_THREE_SPEED
		} else {
			// 前面部分线性从快到慢
			const normalSteps = LOTTERY_TOTAL_JUMPS - 3
			const progress = currentStep / normalSteps
			currentSpeed = LOTTERY_INITIAL_SPEED + (LOTTERY_FINAL_SPEED - LOTTERY_INITIAL_SPEED) * progress
		}

		// 继续下一次跳转
		setTimeout(nextJump, currentSpeed)
	}

	// 开始第一次跳转
	nextJump()
}

// 关闭弹窗
const closeModal = () => {
	showModal.value = false
	currentIndex.value = -1
	winnerIndex.value = -1
}

// 背景音乐控制
const toggleBackgroundMusic = () => {
	if (!backgroundAudio) {
		console.log('请在配置区域设置背景音乐地址')
		return
	}
	
	if (isMusicPlaying.value) {
		backgroundAudio.pause()
		isMusicPlaying.value = false
		console.log('背景音乐已暂停')
	} else {
		backgroundAudio.play().catch(err => console.log('背景音乐播放失败:', err))
		isMusicPlaying.value = true
		console.log('背景音乐已播放')
	}
}

// 切换音量控制面板显示
const toggleVolumeControl = () => {
	showVolumeControl.value = !showVolumeControl.value
}

// 调节背景音乐音量
const adjustMusicVolume = (event: Event) => {
	const target = event.target as HTMLInputElement
	const volume = parseInt(target.value)
	musicVolume.value = volume
	
	if (backgroundAudio) {
		backgroundAudio.volume = volume / 100
		console.log(`背景音乐音量已调节为: ${volume}%`)
	}
}

// 音效播放控制函数

// 播放抽奖音效（每次选中卡片时播放一次）
const playLotterySound = () => {
	try {
		if (LOTTERY_SOUND_URL && lotteryAudio) {
			// 每次播放都从头开始
			lotteryAudio.currentTime = 0
			lotteryAudio.play().catch(err => console.log('抽奖音效播放失败:', err))
		} else {
			// 使用默认的短促音效
			playDefaultBeep()
		}
	} catch (err) {
		console.log('抽奖音效播放错误:', err)
	}
}

// 默认短促音效
const playDefaultBeep = () => {
	try {
		const ctx = initAudioContext()
		const oscillator = ctx.createOscillator()
		const gainNode = ctx.createGain()
		
		oscillator.type = 'sine'
		oscillator.frequency.value = 800
		
		gainNode.gain.setValueAtTime(0.1, ctx.currentTime)
		gainNode.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.1)
		
		oscillator.connect(gainNode)
		gainNode.connect(ctx.destination)
		
		oscillator.start(ctx.currentTime)
		oscillator.stop(ctx.currentTime + 0.1)
	} catch (err) {
		console.log('默认音效播放错误:', err)
	}
}

// 播放中奖音效
const playWinSound = () => {
	try {
		if (WIN_SOUND_URL && winAudio) {
			winAudio.currentTime = 0
			winAudio.play().catch(err => console.log('中奖音效播放失败:', err))
			console.log('中奖音效已播放（自定义）')
		} else {
			// 使用默认的庆祝音效
			playDefaultWinSound()
		}
	} catch (err) {
		console.log('中奖音效播放错误:', err)
	}
}

// 默认中奖音效
const playDefaultWinSound = () => {
	try {
		const ctx = initAudioContext()
		
		const playTone = (frequency: number, startTime: number, duration: number) => {
			const oscillator = ctx.createOscillator()
			const gainNode = ctx.createGain()
			
			oscillator.type = 'sine'
			oscillator.frequency.value = frequency
			
			gainNode.gain.setValueAtTime(0, startTime)
			gainNode.gain.linearRampToValueAtTime(0.3, startTime + 0.01)
			gainNode.gain.exponentialRampToValueAtTime(0.01, startTime + duration)
			
			oscillator.connect(gainNode)
			gainNode.connect(ctx.destination)
			
			oscillator.start(startTime)
			oscillator.stop(startTime + duration)
		}
		
		const now = ctx.currentTime
		playTone(523, now, 0.2)
		playTone(659, now + 0.15, 0.2)
		playTone(784, now + 0.3, 0.3)
		playTone(1047, now + 0.5, 0.4)
		
		console.log('中奖音效已播放（默认）')
	} catch (err) {
		console.log('默认中奖音效播放错误:', err)
	}
}

// 键盘事件处理
const handleKeyPress = (event: KeyboardEvent) => {
	if (event.key === 'Enter' && !isRunning.value && !showModal.value) {
		startLottery()
	}
}

// 组件挂载时添加键盘事件监听
onMounted(() => {
	window.addEventListener('keypress', handleKeyPress)
	// 初始化音频
	initAudio()
})

// 组件卸载时移除键盘事件监听
onUnmounted(() => {
	window.removeEventListener('keypress', handleKeyPress)
	// 清理音频资源
	if (lotteryAudio) {
		lotteryAudio.pause()
		lotteryAudio = null
	}
	if (winAudio) {
		winAudio.pause()
		winAudio = null
	}
	if (backgroundAudio) {
		backgroundAudio.pause()
		backgroundAudio = null
	}
	if (audioContext) {
		audioContext.close()
	}
})
</script>

<style scoped>
.lottery-container {
	max-width: 1400px;
	width: 100%;
	min-height: 100vh;
	padding: 20px;
	position: relative;
	display: flex;
	flex-direction: column;
	justify-content: center;
	align-items: center;
}

/* 背景音乐控制区域 */
.music-control {
	position: fixed;
	top: 20px;
	right: 20px;
	display: flex;
	flex-direction: column;
	align-items: flex-end;
	gap: 10px;
	z-index: 100;
}

/* 按钮组 */
.music-btn-group {
	display: flex;
	gap: 10px;
}

/* 音乐播放按钮 */
.music-btn {
	width: 60px;
	height: 60px;
	font-size: 28px;
	background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
	border: none;
	border-radius: 50%;
	cursor: pointer;
	box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
	transition: all 0.3s ease;
}

.music-btn:hover {
	transform: scale(1.1);
	box-shadow: 0 6px 20px rgba(102, 126, 234, 0.6);
}

.music-btn:active {
	transform: scale(0.95);
}

/* 音量调节按钮 */
.volume-toggle-btn {
	width: 60px;
	height: 60px;
	font-size: 24px;
	background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
	border: none;
	border-radius: 50%;
	cursor: pointer;
	box-shadow: 0 4px 15px rgba(245, 87, 108, 0.4);
	transition: all 0.3s ease;
}

.volume-toggle-btn:hover {
	transform: scale(1.1);
	box-shadow: 0 6px 20px rgba(245, 87, 108, 0.6);
}

.volume-toggle-btn:active {
	transform: scale(0.95);
}

/* 音量控制面板 */
.volume-control {
	display: flex;
	align-items: center;
	gap: 8px;
	padding: 10px 15px;
	background: rgba(102, 126, 234, 0.95);
	border-radius: 25px;
	box-shadow: 0 4px 15px rgba(102, 126, 234, 0.3);
	backdrop-filter: blur(10px);
	animation: slideIn 0.3s ease;
}

@keyframes slideIn {
	from {
		opacity: 0;
		transform: translateX(20px);
	}
	to {
		opacity: 1;
		transform: translateX(0);
	}
}

.volume-icon {
	font-size: 20px;
}

/* 音量滑块 */
.volume-slider {
	width: 120px;
	height: 6px;
	-webkit-appearance: none;
	appearance: none;
	background: rgba(255, 255, 255, 0.3);
	border-radius: 3px;
	outline: none;
	cursor: pointer;
}

.volume-slider::-webkit-slider-thumb {
	-webkit-appearance: none;
	appearance: none;
	width: 16px;
	height: 16px;
	background: #fff;
	border-radius: 50%;
	cursor: pointer;
	box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
	transition: all 0.2s ease;
}

.volume-slider::-webkit-slider-thumb:hover {
	transform: scale(1.2);
	box-shadow: 0 3px 12px rgba(0, 0, 0, 0.4);
}

.volume-slider::-moz-range-thumb {
	width: 16px;
	height: 16px;
	background: #fff;
	border: none;
	border-radius: 50%;
	cursor: pointer;
	box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
	transition: all 0.2s ease;
}

.volume-slider::-moz-range-thumb:hover {
	transform: scale(1.2);
	box-shadow: 0 3px 12px rgba(0, 0, 0, 0.4);
}

/* 音量数值显示 */
.volume-value {
	font-size: 14px;
	color: #fff;
	font-weight: bold;
	min-width: 45px;
	text-align: center;
}

/* 标题区域样式 */
.header {
	text-align: center;
	margin-bottom: 20px;
	position: relative;
}

.title {
	font-size: 36px;
	color: #fff;
	text-shadow: 0 0 20px rgba(255, 215, 0, 0.8),
		0 0 40px rgba(255, 0, 0, 0.6);
	margin-bottom: 8px;
	animation: glow 2s ease-in-out infinite;
}

.title-icon {
	display: inline-block;
	animation: bounce 1s ease-in-out infinite;
}

.subtitle {
	font-size: 20px;
	color: #ffd700;
	letter-spacing: 6px;
	font-weight: bold;
	margin-bottom: 10px;
}

.tip {
	font-size: 16px;
	color: rgba(255, 255, 255, 0.8);
	letter-spacing: 2px;
	animation: tip-blink 2s ease-in-out infinite;
}

@keyframes tip-blink {
	0%, 100% {
		opacity: 0.6;
	}
	50% {
		opacity: 1;
	}
}

@keyframes glow {

	0%,
	100% {
		text-shadow: 0 0 20px rgba(255, 215, 0, 0.8), 0 0 40px rgba(255, 0, 0, 0.6);
	}

	50% {
		text-shadow: 0 0 30px rgba(255, 215, 0, 1), 0 0 60px rgba(255, 0, 0, 0.8);
	}
}

@keyframes bounce {

	0%,
	100% {
		transform: translateY(0);
	}

	50% {
		transform: translateY(-10px);
	}
}

/* 奖品卡片网格布局 */
.prize-grid {
	display: grid;
	grid-template-columns: repeat(4, 1fr);
	gap: 15px;
	margin-bottom: 20px;
	width: 100%;
	max-width: 1200px;
}

/* 奖品卡片样式 */
.prize-card {
	background: linear-gradient(145deg, #fff, #f5f5f5);
	border-radius: 12px;
	padding: 18px 12px;
	text-align: center;
	transition: all 0.3s ease;
	position: relative;
	cursor: pointer;
	box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
	border: 3px solid transparent;
	min-height: 100px;
	display: flex;
	flex-direction: column;
	justify-content: center;
}

.prize-card:hover {
	transform: translateY(-5px);
	box-shadow: 0 8px 25px rgba(0, 0, 0, 0.3);
}

/* 高亮轮廓样式 - 增强瞩目效果 */
.prize-card.highlight {
	border-color: #ffd700;
	border-width: 5px;
	background: linear-gradient(145deg, #fffef0, #fff8dc);
	box-shadow: 0 0 40px rgba(255, 215, 0, 1),
		0 0 60px rgba(255, 215, 0, 0.8),
		0 0 80px rgba(255, 215, 0, 0.6),
		inset 0 0 30px rgba(255, 215, 0, 0.4);
	animation: highlight-pulse 0.4s ease-in-out;
	transform: scale(1.15);
	z-index: 10;
}

/* 中奖卡片样式 */
.prize-card.winner {
	border-color: #ffd700;
	background: linear-gradient(145deg, #fff8dc, #fffaf0);
	box-shadow: 0 0 40px rgba(255, 215, 0, 1),
		0 0 60px rgba(255, 215, 0, 0.6);
	animation: winner-glow 1s ease-in-out infinite;
	transform: scale(1.1);
}

@keyframes highlight-pulse {
	0% {
		transform: scale(1.15);
		box-shadow: 0 0 40px rgba(255, 215, 0, 1),
			0 0 60px rgba(255, 215, 0, 0.8),
			0 0 80px rgba(255, 215, 0, 0.6);
	}

	50% {
		transform: scale(1.2);
		box-shadow: 0 0 50px rgba(255, 215, 0, 1),
			0 0 70px rgba(255, 215, 0, 0.9),
			0 0 100px rgba(255, 215, 0, 0.7);
	}

	100% {
		transform: scale(1.15);
		box-shadow: 0 0 40px rgba(255, 215, 0, 1),
			0 0 60px rgba(255, 215, 0, 0.8),
			0 0 80px rgba(255, 215, 0, 0.6);
	}
}

@keyframes winner-glow {

	0%,
	100% {
		box-shadow: 0 0 40px rgba(255, 215, 0, 1), 0 0 60px rgba(255, 215, 0, 0.6);
	}

	50% {
		box-shadow: 0 0 50px rgba(255, 215, 0, 1), 0 0 80px rgba(255, 215, 0, 0.8);
	}
}

.prize-number {
	position: absolute;
	top: 10px;
	left: 10px;
	width: 30px;
	height: 30px;
	background: #c31432;
	color: #fff;
	border-radius: 50%;
	display: flex;
	align-items: center;
	justify-content: center;
	font-weight: bold;
	font-size: 14px;
	transition: all 0.3s ease;
}

/* 高亮时序号也增强 */
.prize-card.highlight .prize-number {
	background: #ffd700;
	color: #c31432;
	transform: scale(1.3);
	box-shadow: 0 0 15px rgba(255, 215, 0, 0.8);
}

.prize-title {
	font-size: 18px;
	font-weight: bold;
	color: #c31432;
	margin-bottom: 6px;
	transition: all 0.3s ease;
	line-height: 1.3;
}

.prize-desc {
	font-size: 14px;
	color: #666;
	transition: all 0.3s ease;
	line-height: 1.4;
}

/* 高亮时文字也增强 */
.prize-card.highlight .prize-title {
	font-size: 20px;
	color: #ff6b00;
	text-shadow: 0 0 10px rgba(255, 215, 0, 0.5);
}

.prize-card.highlight .prize-desc {
	font-size: 15px;
	color: #333;
	font-weight: 600;
}

/* 抽奖按钮样式 */
.lottery-btn {
	display: block;
	margin: 0 auto;
	padding: 14px 60px;
	font-size: 20px;
	font-weight: bold;
	color: #fff;
	background: linear-gradient(135deg, #c31432, #e74c3c);
	border: none;
	border-radius: 50px;
	cursor: pointer;
	transition: all 0.3s ease;
	box-shadow: 0 8px 25px rgba(195, 20, 50, 0.5);
}

.lottery-btn:hover:not(:disabled) {
	transform: translateY(-3px);
	box-shadow: 0 12px 35px rgba(195, 20, 50, 0.7);
	background: linear-gradient(135deg, #e74c3c, #c31432);
}

.lottery-btn:disabled {
	opacity: 0.7;
	cursor: not-allowed;
	animation: btn-pulse 1s ease-in-out infinite;
}

@keyframes btn-pulse {

	0%,
	100% {
		transform: scale(1);
	}

	50% {
		transform: scale(1.05);
	}
}

/* 弹窗遮罩层 */
.modal-overlay {
	position: fixed;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
	background: rgba(0, 0, 0, 0.7);
	display: flex;
	justify-content: center;
	align-items: center;
	z-index: 1000;
	animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
	from {
		opacity: 0;
	}

	to {
		opacity: 1;
	}
}

/* 弹窗内容 */
.modal-content {
	background: linear-gradient(145deg, #fff, #fffaf0);
	border-radius: 25px;
	padding: 50px 60px;
	max-width: 500px;
	width: 90%;
	text-align: center;
	box-shadow: 0 20px 60px rgba(0, 0, 0, 0.5);
	border: 5px solid #ffd700;
	animation: modalShow 0.5s cubic-bezier(0.68, -0.55, 0.265, 1.55);
}

@keyframes modalShow {
	from {
		transform: scale(0.5) rotate(-5deg);
		opacity: 0;
	}

	to {
		transform: scale(1) rotate(0deg);
		opacity: 1;
	}
}

.modal-header {
	margin-bottom: 30px;
}

.congratulation {
	font-size: 36px;
	font-weight: bold;
	color: #c31432;
	margin-bottom: 15px;
	text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.1);
}

.trophy {
	font-size: 80px;
	animation: trophy-rotate 2s ease-in-out infinite;
}

@keyframes trophy-rotate {

	0%,
	100% {
		transform: rotate(-10deg) scale(1);
	}

	50% {
		transform: rotate(10deg) scale(1.1);
	}
}

.modal-body {
	margin-bottom: 30px;
}

.winner-title {
	font-size: 32px;
	font-weight: bold;
	color: #0088ff;
	margin-bottom: 15px;
	text-shadow: 0 0 10px rgba(255, 215, 0, 0.5);
}

.winner-desc {
	font-size: 24px;
	color: #333;
	font-weight: 500;
}

.close-btn {
	padding: 12px 50px;
	font-size: 20px;
	font-weight: bold;
	color: #fff;
	background: linear-gradient(135deg, #c31432, #e74c3c);
	border: none;
	border-radius: 30px;
	cursor: pointer;
	transition: all 0.3s ease;
	box-shadow: 0 5px 15px rgba(195, 20, 50, 0.4);
}

.close-btn:hover {
	transform: translateY(-2px);
	box-shadow: 0 8px 20px rgba(195, 20, 50, 0.6);
}

/* 响应式设计 */
@media (max-width: 1440px) {
	.lottery-container {
		padding: 15px;
	}
	
	.header {
		margin-bottom: 15px;
	}
	
	.title {
		font-size: 32px;
	}
	
	.subtitle {
		font-size: 18px;
	}
	
	.prize-grid {
		gap: 12px;
		margin-bottom: 15px;
	}
	
	.prize-card {
		padding: 15px 10px;
		min-height: 90px;
	}
	
	.prize-title {
		font-size: 16px;
	}
	
	.prize-desc {
		font-size: 13px;
	}
}

@media (max-width: 1024px) {
	.title {
		font-size: 28px;
	}

	.subtitle {
		font-size: 16px;
	}
	
	.prize-grid {
		gap: 10px;
	}

	.prize-card {
		padding: 12px 8px;
		min-height: 80px;
	}

	.prize-title {
		font-size: 15px;
	}

	.prize-desc {
		font-size: 12px;
	}
	
	.lottery-btn {
		padding: 12px 50px;
		font-size: 18px;
	}
}

@media (max-width: 768px) {
	.lottery-container {
		padding: 10px;
	}
	
	.title {
		font-size: 24px;
	}

	.subtitle {
		font-size: 14px;
		letter-spacing: 4px;
	}
	
	.tip {
		font-size: 14px;
	}

	.prize-grid {
		grid-template-columns: repeat(2, 1fr);
		gap: 10px;
		margin-bottom: 10px;
	}

	.lottery-btn {
		padding: 10px 40px;
		font-size: 16px;
	}
}
</style>
