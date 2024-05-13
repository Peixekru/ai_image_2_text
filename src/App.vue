<!-- Componente de template -->
<template>

	<!-- Componente de loading -->
	<Loading ref="isLoading" />

	<!-- Contêiner principal -->
	<div class="group-container">

		<!-- Condição para exibir o ícone de início -->
		<div v-if="resultText == ''">
			<!-- Ícone de início desligado se nenhum arquivo estiver selecionado -->
			<IconStart v-if="selectedFile == null" 
			class="fadeIn"
			:icon="toggleOff" size="2x" color="grey" 
			/>
			<!-- Ícone de início ligado se um arquivo estiver selecionado -->
			<IconStart v-else 
			class="fadeIn"
			:icon="toggleOn" size="2x" color="blueviolet" />
		</div>

		<!-- Título exibido quando há texto resultante -->
		<h2 v-if="resultText != ''" class="title-box">
			<span class="icon-pos fadeIn">
				<!-- Ícone de marca de seleção -->
				<IconStart :icon="checkIcon" size="1x" color="green" />
			</span>
			Sua imagem em texto!
		</h2>

		<!-- Área de soltar arquivo -->
		<div v-if="resultText == ''" class="drop-area fadeIn" @dragover.prevent @drop="handleDrop">
			<!-- Mostra a imagem se a URL estiver disponível, senão exibe um texto de placeholder -->
			<img v-if="imageUrl" class="img-placeholder" :src="imageUrl" alt="Imagem">
			<span v-else class="placeholder-txt">Arraste e solte sua imagem aqui!</span>
		</div>

		<!-- Botão para converter imagem em texto -->
		<button v-if="imageUrl != null" class="form-btn-send fadeIn" @click="run">
			Converter para texto
		</button>

		<!-- Texto resultante da conversão -->
		<p v-if="resultText" class="result-text fadeIn">{{ resultText }}</p>

		<!-- Botão para iniciar nova conversão -->
		<button v-if="imageUrl == null && resultText != ''" class="form-btn-clear" @click="clearResult">
			Nova conversão
		</button>

	</div>
</template>

<!-- Configuração do script -->
<script setup>

import { ref } from 'vue';

// Importa o GoogleGenerativeAI e suas configurações
import { GoogleGenerativeAI, HarmCategory, HarmBlockThreshold } from "@google/generative-ai";

// Referência para o componente de loading
const isLoading = ref(null);

// Funções para iniciar e parar o loading
const startLoading = () => {
	isLoading.value.showLoading();
}

const stopLoading = () => {
	isLoading.value.hideLoading();
}

// Importação do componente de loading
import Loading from "./components/Loading.vue"

// Importação de ícones
import IconStart from "./components/icons/IconStart.vue"
import {
	faCheck,
	faToggleOn,
	faToggleOff,
} from '@fortawesome/free-solid-svg-icons';

// Atribuição de ícones a variáveis
const checkIcon = faCheck;
const toggleOn = faToggleOn;
const toggleOff = faToggleOff;

// Declaração de variáveis reativas
const selectedFile = ref(null);
const resultText = ref('');
const imageUrl = ref(null);

// Função para limpar os resultados e reiniciar o processo
const clearResult = () => {
	selectedFile.value = null;
	resultText.value = '';
	imageUrl.value = null;
}

// Configurações do modelo e chave da API
const MODEL_NAME = "gemini-1.0-pro-vision-latest";
const API_KEY = import.meta.env.VITE_API_KEY;

// Função para lidar com o evento de soltar uma imagem na área designada
const handleDrop = (event) => {
	event.preventDefault();
	const file = event.dataTransfer.files[0];
	selectedFile.value = file;
	// Verifica se o arquivo é uma imagem
	if (file.type.startsWith('image/')) {
		imageUrl.value = URL.createObjectURL(file);
	} else {
		alert('Por favor, selecione um arquivo de imagem.');
	}
}

// Função para executar a conversão da imagem em texto
const run = async () => {

	// Inicia o loading
	startLoading();

	const genAI = new GoogleGenerativeAI(API_KEY);
	const model = genAI.getGenerativeModel({ model: MODEL_NAME });

	// Configurações de geração de texto
	const generationConfig = {
		temperature: 0.5,
		topK: 35,
		topP: 1,
		maxOutputTokens: 4096,
	};

	// Configurações de segurança
	const safetySettings = [
		{
			category: HarmCategory.HARM_CATEGORY_HARASSMENT,
			threshold: HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
		},
		{
			category: HarmCategory.HARM_CATEGORY_HATE_SPEECH,
			threshold: HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
		},
		{
			category: HarmCategory.HARM_CATEGORY_SEXUALLY_EXPLICIT,
			threshold: HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
		},
		{
			category: HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT,
			threshold: HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
		},
	];

	// Verifica se um arquivo foi selecionado
	if (!selectedFile.value) {
		throw new Error("Please select an image file.");
	}

	// Lê o arquivo como URL
	const reader = new FileReader();
	reader.readAsDataURL(selectedFile.value);
	reader.onload = async () => {
		const parts = [
			{ text: "Converta a imagem em texto. Envie a resposta JSON, utilizando a chave 'msg' " },
			{
				inlineData: {
					mimeType: "image/jpeg",
					data: reader.result.split(',')[1]
				}
			},
		];

		const result = await model.generateContent({
			contents: [{ role: "user", parts }],
			generationConfig,
			safetySettings,
		});

		const response = result.response;

		// Manipula a resposta JSON e atualiza o valor de resultText
		const correctedJsonData = await response.text().slice(8, -3).replace(/`/g, "");
		const jsonObject = JSON.parse(correctedJsonData);;

		// Insere o texto recebido na variável 
		resultText.value = jsonObject.msg;

		// Remove a imagem da área designada
		imageUrl.value = null;

		// Para o loading
		stopLoading();
	}
}

// Funções para mostrar e ocultar o componente de loading
const showLoading = () => {
	loadingVisible.value = true;
}

const hideLoading = () => {
	loadingVisible.value = false;
}

// Referência para a visibilidade do loading
const loadingVisible = ref(false);

</script>

<!-- Estilos CSS -->
<style>
.drop-area {
	width: 80vw;
	height: 140px;
	border: 2px dashed rgba(255, 255, 255, .3);
	text-align: center;
	border-radius: 1em;
	cursor: pointer;
	margin: 2em 0;
	padding: 1em .5em;
	display: flex;
	align-items: center;
	justify-content: center;
}

.drop-area img {
	max-width: 100%;
	max-height: 100%;
}

button {
	margin-bottom: 2em;
	width: 80vw;
}

.form-btn-send {
	background-color: blueviolet;
}

.form-btn-clear {
	background-color: rgb(239, 62, 77);
}

.group-container {
	width: 80vw;
	display: flex;
	flex-direction: column;
	align-items: center;
}

.result-text {
	width: 80vw;
	border-top: 1px solid rgba(255, 255, 255, .3);
	padding: 2em 0;
	margin-top: 2em;
	text-align: left;
	font-size: .9em;
}

.placeholder-txt {
	color: rgba(255, 255, 255, .5);
}

.title-box {
	width: 80vw;
	text-align: left;
	margin: 0;
}

.img-placeholder {
	border-radius: 8px;
}

.icon-pos {
	margin-right: 0.4em;
}

/* Definição da animação fadeIn */
@keyframes fadeIn {
	0% {
		opacity: 0;
		/* Inicia com opacidade zero */
	}

	100% {
		opacity: 1;
		/* Termina com opacidade máxima */
	}
}

/* Aplicação da animação fadeIn a um elemento */
.fadeIn {
	animation: fadeIn 1s ease-in-out;
	/* Duração de 0.5 segundos com transição suave */
}

</style>
