<template>
	<view  style="position: relative; width: 100vw; height: 100vh;">
		<view>
			<video
				class="fullscreen-video position-absolute top-0 left-0"
				autoplay
				http-cache
				:style="videoStyle"
				:src="currentVideo === 'question' ? questionVideoPath : transitionPath"
				:loop="currentVideo === 'transition'"
				:controls="false"
				:show-loading="false"
				:show-center-play-btn="false"
				:show-play-btn="false"
				@ended="videoEnded"
				@loadedmetadata="handleMetadataLoaded"
			></video>
			<view 
			    class="video-mask position-absolute top-0 left-0"
				:style="videoStyle"
			    @touchmove.prevent
			></view>
		</view>
		<!-- 使用同一个位置和大小的容器来显示摄像头或“关闭中”提示 -->
		<view class="camera-wrapper">
			<camera v-if="isCameraActive" device-position="front"></camera>
			<view v-else class="closed-view">摄像头关闭中</view>
		</view> 
		
		<view class="bottom-body padding-xl">
			<view class="question-body border-radius-large">
				<text class="question-text">{{ questionText }}</text>
			</view>
			<view class="btn-wrapper">
				<view 
					class="iconfont toggle-camera-btn" 
					:class="isCameraActive ? 'icon-luxiangguan' : 'icon-luxiang1'" 
					@tap="toggleCamera"
				></view>
				<!-- 根据是否回答中显示不同按钮 -->
				<view class="btn-box">
					<u-button
						v-if="!isAnswering"
						shape="circle" 
						type="primary" 
						size="large" 
						:text="isInterviewFinished ? '结束面试' : '开始答题'"
						:disabled="!canAnswer"
						@click="startAnswering" 
					></u-button>
					<u-button
						v-else
						shape="circle" 
						type="primary" 
						size="large" 
						:text="`回答完毕 ( ${formattedCountdown} )`"
						:disabled="isButtonDisabled"
						@click="stopAnswering" 
					></u-button>
				</view>
			</view>
		</view>
		<view class="bottom-bg position-absolute bottom-0 left-0 right-0"></view>
	</view>
</template>


<script setup lang="ts">
	import { reactive, ref, toRefs, computed, onMounted, onBeforeUnmount } from 'vue';
	import { fetchIvQuestion, submitInterviewAnswer, audioToText } from '@/api/home';
	import { useInterviewStore } from '@/stores';
	const interviewStore = useInterviewStore();
	const currentJobInfo = computed(() => interviewStore.currentJobInfo);
	const flowNavTitle = currentJobInfo.value ? currentJobInfo.value.name : '模拟面试';
	const ivCustomParams = computed(() => interviewStore.ivCustomParams);
	
	const CONFIG = {
	    INITIAL_COUNTDOWN: 300,
		PROLOGUE_VIDEO_PATH: "https://mp-43f7552d-29af-4d0a-8672-7a2fcdd00dc7.cdn.bspapp.com/interview/iv-prologue.mp4",
		PROLOGUE_VIDEO_TEXT: "您好！欢迎使用我们的AI模拟面试应用。我将会是您今日的面试官。我们致力于为您提供一个公正和真实的面试体验，以帮助您更好地准备实际面试。在开始之前，能请您简短地介绍一下自己吗？",
	    TRANSITION_VIDEO_PATH: "https://mp-43f7552d-29af-4d0a-8672-7a2fcdd00dc7.cdn.bspapp.com/interview/iv-transition.mp4",
	    END_VIDEO_PATH: "https://mp-43f7552d-29af-4d0a-8672-7a2fcdd00dc7.cdn.bspapp.com/interview/iv-end-video.mp4",
		END_VIDEO_TEXT: "您的本次面试已经顺利结束，感谢您的参与与努力。请您稍等片刻，我们正在为您生成专属的面试报告。稍后，您可以在'面试历史'里查看您的面试表现和反馈。再次感谢您，祝您一切顺利！"
		// ... 其他硬编码的值
	};
	
	const RECORDER_OPTIONS = {
		duration: 300000,
		sampleRate: 16000,
		numberOfChannels: 1,
		encodeBitRate: 48000,
		format: 'mp3',
	};

	const myData = reactive({
	    // 视频相关属性
	    video: {
	        recordVideoPath: '',
	        isCameraActive: true, // 预览视频开启状态
	        isRecording: false, // 是否在录制视频中
	        transitionPath: CONFIG.TRANSITION_VIDEO_PATH,
	        questionVideoPath: "",
	        currentVideo: 'question', // 'question', 'transition'
	    },
	
	    // 面试流程属性
	    interview: {
	        isAnswering: false,
	        canAnswer: false,
	        countdown: CONFIG.INITIAL_COUNTDOWN,
	        countdownInterval: null as number | null,
	        questionText: "",
	        isInterviewFinished: false,
	        questions: [] as Questions[],  // 存储所有的面试问题
	        currentQuestionIndex: 0,  // 当前问题的索引
			userAnswers: [] as UserAnswer[],  // 存储用户的回答
	    },
	
	    // 界面/显示属性
	    display: {
	        isButtonDisabled: true,
	        sysWidth: 0,
	        sysHeight: 0,
	        videoWidth: 0,
	        videoHeight: 0
	    }
	});
	const {
	    recordVideoPath,
	    isCameraActive,
		isRecording,
		transitionPath,
		questionVideoPath,
		currentVideo
	} = toRefs(myData.video);
	
	const {
	    isAnswering,
	    canAnswer,
	    countdown,
	    countdownInterval,
	    questionText,
	    isInterviewFinished,
	    questions,
	    currentQuestionIndex,
		userAnswers
	    // ... 其他面试相关属性
	} = toRefs(myData.interview);
	
	const {
	    isButtonDisabled,
	    sysWidth,
	    sysHeight,
	    videoWidth,
	    videoHeight
	} = toRefs(myData.display);
	
	const cameraCtx = uni.createCameraContext();
	
	// 计算后的视频宽高
	const videoStyle = computed(() => {
	    return {
	        width: `${videoWidth.value}px`,
	        height: `${videoHeight.value}px`
	    };
	});
	
	// 自定义导航标题
	const setCustomNavigationBarTitle = () => {
		uni.setNavigationBarTitle({
			title: flowNavTitle || "模拟面试"
		})
	}
	
	// 倒计时格式化
	const formattedCountdown = computed(() => {
		const minutes = Math.floor(countdown.value / 60);
		const seconds = countdown.value % 60;
		return `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
	});
	
	// 倒计时方法
	const startCountdown = () => {
		countdownInterval.value = setInterval(() => {
			if (countdown.value > 0) {
				countdown.value--;
			} else {
				clearInterval(countdownInterval.value!);
				stopAnswering();
			}
		}, 1000);
	};
	
	// 当视频播放结束时，根据当前视频类型进行相应的操作
	const videoEnded = () => {
		if (currentVideo.value === 'question') {
			switchToTransitionVideo();
		} else if (currentVideo.value === 'transition') {
			// ... 其他逻辑
		}
	};
	
	// 切换到过渡视频并更新相关状态
	const switchToTransitionVideo = () => {
	    currentVideo.value = 'transition';
	    canAnswer.value = true;
	    isButtonDisabled.value = false;
	};

	// 开始答题
	const startAnswering = () => {
		if(isInterviewFinished.value) {
			vk.switchTab("/pages/home/index");
		}else {
			
			isAnswering.value = true;
			isButtonDisabled.value = false;
			canAnswer.value = false;
			startCountdown();
			if (isCameraActive.value) {
			    startRecord();
			}
			startRecordingAudio();
		}
		
	}
	
	// 是否为最后一个面试问题
	const isLastQuestion = computed(() => currentQuestionIndex.value === questions.value.length - 1);

	// 回答完毕按钮点击事件
	const stopAnswering = async () => {
		try {
			if (countdownInterval.value) {
			    clearInterval(countdownInterval.value);
			    countdown.value = 300; // 重置计时器
			}
			isAnswering.value = false;
			
			// 收集用户的回答
			const currentQuestion: Questions = questions.value[currentQuestionIndex.value];
			const userAnswer = {
				question_id: currentQuestion.id, // 假设每个问题对象都有一个唯一的ID
				answer: questionText.value, // 这里假设questionText是用户的回答
				recording_url: "录音文件URL", // 这里需要你的录音逻辑来提供真实的URL
				video_url: recordVideoPath.value // 这是录制的视频的临时路径
			}; 
			userAnswers.value.push(userAnswer);
				
			if (isLastQuestion.value) {
			    // 所有问题都已回答
			    showEndVideo();
			} else {
			    nextQuestion(); // 加载下一个问题
			}
				
			if (isRecording.value) {
			    await stopRecord();
			    uploadVideo(recordVideoPath.value);
			}
			stopRecordingAudio(); 
		}catch(err){
			console.error('Error while stopping the answer:', err);
		}
	    
	};
	
	// 随机获取5道面试题数据
	const fetchInterviewQuestions = async () => {
		try {
			const res = await fetchIvQuestion(ivCustomParams.value);
			let resQuestions = res.data || []
			resQuestions.unshift({
				video: CONFIG.PROLOGUE_VIDEO_PATH,
				text: CONFIG.PROLOGUE_VIDEO_TEXT
			})
			
			if (resQuestions.length) {
				questions.value = resQuestions;
				questionVideoPath.value = resQuestions[0].video;
				questionText.value = resQuestions[0].text;
			} else {
				console.error("没有获取到面试问题");
			}
		} catch (error) {
			console.error('Error during fetchInterviewQuestions', error);
			// 可以在这里添加更多的错误处理逻辑，比如设置一个标志，让用户知道出现了错误
		}
	};
	
	// 展示下一个问题
	const nextQuestion = () => {
		console.log("nextQuestion method called");  // 添加这一行
	    currentQuestionIndex.value++;
	    
	    if (currentQuestionIndex.value < questions.value.length) {
	        const currentQuestion = questions.value[currentQuestionIndex.value];
	        
	        if (currentQuestion) {
	            questionVideoPath.value = currentQuestion.video;
	            questionText.value = currentQuestion.text;
	            currentVideo.value = 'question';  // 这里确保设置为播放问题视频
	        } else {
	            console.error("当前问题索引超出问题数组范围");
	        }
	    } else {
	        // 所有问题都已回答
	        showEndVideo();
	    }
	}
	
	// 展示结束视频并通知用户报告正在生成
	const showEndVideo = () => {
		isInterviewFinished.value = true;
		canAnswer.value = true;
		currentVideo.value = 'question';

	    // 设置结束视频
	    questionVideoPath.value = CONFIG.END_VIDEO_PATH;
	    questionText.value = CONFIG.END_VIDEO_TEXT;
		
		// 上传用户的回答到后端
		uploadUserAnswersApi();
	}
	
	// 上传用户的回答API
	const uploadUserAnswersApi = async () => {
		const params: InterviewAnswer = {
			currentJobInfo: currentJobInfo.value,
			ivCustomParams: ivCustomParams.value,
			answers: userAnswers.value || []
		}
	    try {
			const res = await submitInterviewAnswer(params);
			console.log(res, 'uploadUserAnswersApi');
	    } catch (error) {
	        console.error('Error uploading user answers:', error);
	    }
	};
	
	// 上传视频到服务器
	const uploadVideo = (recordVideoPath: string) => {
		// 这里是您上传视频到服务器的逻辑
		console.log(`上传视频: ${recordVideoPath}`);
	}
	
	// 视频录制预览开关
	const toggleCamera = () => {
	    if (isAnswering.value) {
	        vk.toast("正在回答问题，不能切换摄像头");
	        return;  // 如果正在回答问题，直接返回，不执行后面的代码
	    }
	    isCameraActive.value = !isCameraActive.value;  // 切换摄像头状态
	    if (isCameraActive.value) {
	        startRecord()
	    } else {
	        if (isRecording.value) {  // 只有当摄像头正在录制时才尝试停止录制
	            stopRecord();
	        }
	    }
	}

	// 开始录制视频
	const startRecord = () => {
		cameraCtx.startRecord({
			success: () => {
				console.log('开始录制');
			},
			fail: (err) => {
				console.error('录制失败:', err);
			}
		});
		isRecording.value = true;
	};

	// 停止录制视频
	const stopRecord = async () => {
	    try {
	        const res = await new Promise<any>((resolve, reject) => {
	            cameraCtx.stopRecord({
	                success: resolve,
	                fail: reject
	            });
	        });
	
	        console.log('录制完成', res.tempVideoPath);
			isRecording.value = false;  // 更新录制状态
	        recordVideoPath.value = res.tempVideoPath;
	
	        // 其他处理逻辑...
	
	        return res;  // 返回录制的结果，如果需要的话
	
	    } catch (err) {
	        console.error('停止录制失败:', err);
	        throw err;  // 抛出错误，以便在调用函数中进行进一步处理
	    }
	};
	
	// 获取当前设备宽度
	const getSystemDimensions = () => {
	    const systemInfo = uni.getSystemInfoSync();
	    sysWidth.value = systemInfo.safeArea!.width;
	    sysHeight.value = systemInfo.safeArea!.height;
	}
	
	// 获取视频的宽高，并且和设备宽度做计算，防止视频出现黑边
	const handleMetadataLoaded = (event: any) => {
	    const { width, height } = event.detail;
	
	    const videoAspectRatio = width / height;
	    const deviceAspectRatio = sysWidth.value / sysHeight.value;
	
	    if (videoAspectRatio > deviceAspectRatio) {
	        videoWidth.value = sysWidth.value;
	        videoHeight.value = Math.round(sysWidth.value / videoAspectRatio);
	    } else {
	        videoHeight.value = sysHeight.value;
	        videoWidth.value = Math.round(sysHeight.value * videoAspectRatio);
	    }
	}
	
	// 获取全局唯一的录音管理器
	const recorderManager = uni.getRecorderManager();
	
	// 开始录制音频
	const startRecordingAudio = () => {
	    recorderManager.start(RECORDER_OPTIONS);
	};
	
	// 结束录制音频
	const stopRecordingAudio = () => {
	    recorderManager.stop();
	};
	
	// 监听录音停止，获取临时文件路径
	const onStopRecordingAudio = () => {
		recorderManager.onStop((res) => {
		    const { tempFilePath } = res;
			vk.uploadFile({
				filePath: tempFilePath,
				success: (res: any) => {
					const { url } = res;
					audioToTextApi(url);
				},
				fail(err: any) {
					console.log('Error:', err);
				},
				complete(complete: any) {
					console.log(complete, 'complete')
				}
			});
		});
	}
	
	// 百度云语音转文字接口
	const audioToTextApi = async (filePath: string) => {
		const params = {
			filePath: filePath
		}
	    const res = await audioToText(params);
		console.log(res, 'audioToTextApi')
	};

	onMounted(async () => {
		setCustomNavigationBarTitle();
		getSystemDimensions();
		await fetchInterviewQuestions();
		onStopRecordingAudio();
	});
	
	// 当组件卸载时，清除计时器
	onBeforeUnmount(() => {
	    if (countdownInterval.value) {
	        clearInterval(countdownInterval.value);
	    }
	});
</script>

<style lang="scss" scoped>
	@import '@/common/css/mixins.scss';
	
	.camera-wrapper {
		position: absolute;  // 使用绝对定位
		top: 30rpx;
		left: 30rpx;
		width: 160rpx;
		height: 220rpx;
		overflow: hidden; // 确保摄像头的预览或“关闭中”的提示不超出容器的边界
		background-color: #eee;  // 给容器一个背景色，以确保在摄像头关闭时有一个背景

		camera {
			width: 100%;   // 宽度和高度为100%确保填充整个容器
			height: 100%;
		}

		.closed-view {
			width: 100%;
			height: 100%;
			display: flex;
			align-items: center;
			justify-content: center;
			background-color: rgba(0, 0, 0, 0.5);  // 为“关闭中”的背景添加一个半透明的黑色，使文字更容易阅读
			font-size: 12px;  // 根据需要调整字体大小
		}
	}
	.bottom-body {
		position: absolute;
		bottom: 50rpx;
		left: 0;
		right: 0;
		z-index: 3;
		.question-body {
			min-height: 240rpx;
			padding: 30rpx;
			background-color: rgba(43, 44, 48, 0.9);
			margin-bottom: 70rpx;
			.question-text {
				line-height: 1.7em;
				color: #fff;
				font-size: 28rpx;
			}
		}
		.btn-wrapper {
			@include flex-layout(between);
			.toggle-camera-btn {
				width: 100rpx;
				height: 100rpx;
				border-radius: 50%;
				overflow: hidden;
				color: #fff;
				background-color: $uni-color-warning;
				box-sizing: border-box;
				font-size: 60rpx;
				@include flex-layout(center);
			}
			.btn-box {
				width: 540rpx;
			}
		}
	}
	
	.fullscreen-video {
		object-fit: cover;
		z-index: 0;  
	}
	
	.video-mask {
	    z-index: 1;  // 确保遮罩层在视频之上
	    background-color: rgba(0, 0, 0, 0);  // 将其设置为极低的不透明度，从而能够捕获事件，但对用户来说几乎是透明的
	}
	.bottom-bg {
		background: linear-gradient(to top, #35363a 0%, #3d4144 70%, rgba(61, 65, 68, 0.5) 85%, rgba(61, 65, 68, 0) 100%);
		height: 300rpx;
		z-index: 2;
	}
</style>
