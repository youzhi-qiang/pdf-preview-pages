<template>
	<div class="box">
		<div class="home">
			<span class="title">pdf链接:</span>
			<el-input v-model="inputValue" placeholder="请输入pdf链接"></el-input>
		<el-button type="primary" @click="handleConfirm">确认</el-button>

	    </div>
    	<PDFPreview :url="inputValue"></PDFPreview>
	</div>

 
</template>

<script lang="ts" setup>
import { ref, onMounted } from 'vue'
import PDFPreview from '@/views/main/components/PDFPreview.vue'
const inputValue = ref('https://501351981.github.io/vue-office/examples/dist/static/test-files/test.pdf')
const dialogVisible = ref(false)

const handleConfirm = () => {
	dialogVisible.value = true
}
</script>
<style scoped>
.box{
	width: 91vw;
    height: 87vh;
	overflow:hidden;
	margin: 0 auto;
}

.home {
    width: 80%;
	display: flex;
	flex-direction: row;
	justify-content: center;
	gap:10px;
	align-items: center;
	margin-bottom:20px

}
.title{
    font-size: 20px;
    font-weight: 700;
	white-space: nowrap;
	
}
</style>