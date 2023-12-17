<template lang="pug">
.root
	.image-container(ref="imageContainer")
		//video(ref="videoPlayer" id="test_video" controls :src="streamURL")
		img.img(src="/img/test.jpg" ref="sourceImage" @load="$_edit_setSettings")
		svg(ref="svgContainer"
			class="svg-container"
			:width="settings.svgWidth"
			:height="settings.svgHeight"
			@mousedown="$_edit_startRectangle"
			@mousemove="$_edit_drawRectangle"
			@mousemove.passive="$_edit_moveRectangle"
			@mouseup.passive="isDraggable = false"
			@mouseup="$_edit_endRectangle")

	.controls
		button(v-if="isEditing" @click="$_edit_endEditing" ) Закрыть редактирование
		button(@click="$_edit_clearRectangles") Очистить разметку
		button(@click="$_edit_submitfinalData") Отправить
	.list
		.list-svg(v-for="(polygon, index) in rectangles" :key="index")
			.title Разметка {{index + 1}}
			button.edit(@click="$_edit_deleteRectangle(index)") Удалить
			button.edit(@click="$_edit_startEditing(index)") Редактировать
</template>

<script>
	import SVG from 'svg.js'
	export default {
		data() {
			return {
				settings: {
					naturalWidth: 0,
					naturalHeight: 0,
					svgWidth:800,
					svgHeight: 450,
					ratio: 0
				},
				svg: null,
				drawing: null,
				rectangles: [],
				currentRectangle: null,
				// edit
				editingRectangleIndex: null, // индекс редактируемого полигона
				editingPoints: [], // точки
				isEditing: false, // активно редактирование у полигона
				isDraggable: false, // удерживают за точку или нет
				dragStartX: 0, // координаты точек редактирования
				dragStartY: 0,
			};
		},
		methods: {
			$_edit_getData(){
				let point = {}
				const json = [
					[ 512, 836, 717, 301 ],
					[ 701, 788, 621, 218 ],
					[ 884, 737, 541, 154 ]
				]
				json.forEach(rect => {
					point.startX = Math.ceil(rect[0] / this.settings.naturalWidth * this.settings.svgWidth)
					point.startY = Math.ceil(rect[1] / this.settings.naturalHeight * this.settings.svgHeight)
					point.width = Math.ceil(rect[2] / this.settings.naturalWidth * this.settings.svgWidth)
					point.height = Math.ceil(rect[3] / this.settings.naturalHeight * this.settings.svgHeight)
					point.strokeColor = "#12FF0D"

					this.rectangles.push(point)
					point = {}
				})


				this.$_edit_renderRectangles()
			},
			$_edit_submitfinalData(){
				const data = []
				let point = []
				this.rectangles.forEach(rect => {
					point[0] = Math.ceil(rect.startX * this.settings.naturalWidth / this.settings.svgWidth)
					point[1] = Math.ceil(rect.startY * this.settings.naturalHeight / this.settings.svgHeight)
					point[2] = Math.ceil(rect.width * this.settings.naturalWidth / this.settings.svgWidth)
					point[3] = Math.ceil(rect.height * this.settings.naturalHeight / this.settings.svgHeight)
					data.push(point)
					point = []
				})
				console.log(data)
				// отправка на сервер
			},
			$_edit_setSettings(){
				const image = this.$refs.sourceImage
				this.settings.naturalWidth = image.naturalWidth
				this.settings.naturalHeight = image.naturalHeight
				this.settings.svgWidth = image.width
				this.settings.svgHeight = image.height

				this.$_edit_setupDrawing();
				this.$_edit_getData()
			},
			$_edit_setupDrawing() {
				this.svg = SVG(this.$refs.svgContainer).size(this.settings.svgWidth, this.settings.svgHeight)
				this.drawing = this.svg.group();
			},
			$_edit_startRectangle(event) {
				if(!this.isEditing){
					const { x, y } = this.$_edit_getMousePosition(event)
					this.currentRectangle = {
						startX: x,
						startY: y,
						width: 0,
						height: 0,
						strokeColor: "#12FF0D"
					};
				}
			},
			$_edit_drawRectangle(event) {
				if(this.currentRectangle&&!this.isEditing){
					const { x, y } = this.$_edit_getMousePosition(event)
					this.currentRectangle.width = x - this.currentRectangle.startX
					this.currentRectangle.height = y - this.currentRectangle.startY

					this.$_edit_renderRectangles();
				}
			},
			$_edit_endRectangle() {
				console.log('WORKED END')
				if (this.currentRectangle&&!this.isEditing) {
					this.rectangles.push({...this.currentRectangle});
					this.currentRectangle = null
				}
				// if(this.isEditing) this.endEditing()
			},
			$_edit_renderRectangles() {
				console.log('WORKED RENDER')
				this.svg.clear();
				this.rectangles.forEach((rectangle) => {
					this.svg.rect(rectangle.width, rectangle.height)
						.move(rectangle.startX, rectangle.startY)
						.stroke({ color: '#12FF0D', width: 2 })
						.fill("transparent")
				});

				if (this.currentRectangle) {
					const { startX, startY, width, height } = this.currentRectangle;
					this.svg.rect(width, height)
						.move(startX, startY)
						.stroke({ color: '#f2ff00', width: 2 })
						.fill("transparent")
				}
				if(this.isEditing){
					// добавление точек для редактирования
					this.editingPoints.forEach((point, idx) => {
						this.svg
							.circle(6)
							.fill('#fff')
							.stroke('#12FF0D')
							.move(point.x, point.y)
							.on('mousedown', () => {
								console.log(idx)
								this.isDraggable = true
								// начальные координаты точки для изменения размеров
								this.dragStartX = this.editingPoints[idx].x
								this.dragStartY = this.editingPoints[idx].y
								console.log('START DOWN POINT ..........')
							})
					})
				}
			},
			$_edit_deleteRectangle(index){
				this.rectangles.splice(index, 1)
				this.$_edit_renderRectangles()
			},
			$_edit_clearRectangles() {
				this.rectangles = []
				this.$_edit_endEditing()
				this.currentRectangle = null
				this.$_edit_renderRectangles()
			},
			$_edit_getMousePosition(event) {
				const CTM = this.svg.node.getScreenCTM();
				return {
					x: (event.clientX - CTM.e) / CTM.a,
					y: (event.clientY - CTM.f) / CTM.d,
				};
			},
			$_edit_startEditing(index){
				this.editingRectangleIndex = index;
				this.editingPoints = this.$_edit_calculateEditingPoints(this.rectangles[index])
				console.log(this.editingPoints)
				this.isEditing = true
				this.$_edit_renderRectangles()
			},
			$_edit_startEditingPoint(pointIndex){
				// Запоминаем начальные координаты точки для изменения размеров
				this.dragStartX = this.editingPoints[pointIndex].x
				this.dragStartY = this.editingPoints[pointIndex].y
				this.isEditing = true
			},
			$_edit_calculateEditingPoints(rectangle) {
				const halfWidht = 3 // половина ширины точки
				const points = []
				points.push({ x: rectangle.startX, y: rectangle.startY })
				points.push({ x: rectangle.startX + rectangle.width, y: rectangle.startY })
				points.push({ x: rectangle.startX, y: rectangle.startY + rectangle.height })
				points.push({ x: rectangle.startX + rectangle.width, y: rectangle.startY + rectangle.height })
				points.forEach(point => {
					point.x -= halfWidht
					point.y -= halfWidht
				})
				return points;
			},
			$_edit_calculateSideMove(rectangle, mouseX, mouseY){
				const spread = 10 // разброс
				const x = rectangle.startX
				const y = rectangle.startY

				if (
					mouseX >= x - spread &&
					mouseX <= x + spread &&
					mouseY >= y - spread &&
					mouseY <= y + spread
				) {
					return "top-left"
				} else if (
					mouseX >= x + rectangle.width - spread &&
					mouseX <= x + rectangle.width + spread &&
					mouseY >= y - spread &&
					mouseY <= y + spread
				) {
					return "top-right"
				} else if (
					mouseX >= x - spread &&
					mouseX <= x + spread &&
					mouseY >= y + rectangle.height - spread &&
					mouseY <= y + rectangle.height + spread
				) {
					return "bottom-left"
				} else if (
					mouseX >= x + rectangle.width - spread &&
					mouseX <= x + rectangle.width + spread &&
					mouseY >= y + rectangle.height - spread &&
					mouseY <= y + rectangle.height + spread
				) {
					return "bottom-right"
				}

				return null
			},
			$_edit_endEditing() {
				this.isEditing = false;
				this.isDraggable = false
				this.editingRectangleIndex = null;
				this.editingPoints = []
				this.$_edit_renderRectangles()
			},
			$_edit_moveRectangle(event) {
				if (this.isDraggable && this.editingRectangleIndex !== null) {
					const { x, y } = this.$_edit_getMousePosition(event);
					const rectangle = this.rectangles[this.editingRectangleIndex]

					// Изменения размеров
					const dx = x - this.dragStartX
					const dy = y - this.dragStartY

					// Определяю точку, за которую тянем
					const editingPoint = this.$_edit_calculateSideMove(this.rectangles[this.editingRectangleIndex], x, y)

					// Изменяю размеры прямоугольника в зависимости от точки
					if (this.editingRectangleIndex !== null && editingPoint !== null) {
						switch (editingPoint) {
							case "top-left":
								rectangle.startX += dx
								rectangle.startY += dy
								rectangle.width -= dx
								rectangle.height -= dy
								break;
							case "top-right":
								rectangle.startY += dy
								rectangle.width += dx
								rectangle.height -= dy
								break;
							case "bottom-left":
								rectangle.startX += dx
								rectangle.width -= dx
								rectangle.height += dy
								break;
							case "bottom-right":
								rectangle.width += dx
								rectangle.height += dy
								break;
						}

						this.editingPoints = this.$_edit_calculateEditingPoints(rectangle);

						this.dragStartX = x
						this.dragStartY = y

						this.$_edit_renderRectangles()
					}
				}
			},
		},
	};
</script>

<style scoped>
	.root {
		width: 100vw;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 50px;
	}
	.root video{
		width: 80%;
		max-width: 80%;
	}
	/*.root .video{*/
	/*	width: 800px;*/
	/*	max-width: 800px;*/
	/*}*/
	/*.root .video img{*/
	/*	max-width: 100%;*/
	/*}*/
	.image-container {
		position: relative;
		width: 100%;
		max-width: 800px; /* Adjust as needed */
		margin: 0 auto;
		overflow: hidden;
		height: fit-content;
	}
	.img{
		max-width: 100%;
		pointer-events: none;
	}
	.svg-container {
		position: absolute;
		top: 0;
		left: 0;
		z-index: 10;
		height: 100%;
		width: 100%;
		max-width: 100%;
		max-height: 100%;
	}
	.root{
		width: 100%;
		display: flex;
		justify-content: center;
		align-items: center;
		flex-direction: column;
		row-gap: 50px;
	}
	.root .controls{
		display: flex;
		column-gap: 20px;
	}
	.root .list{
		width: 100%;
		display: grid;
		grid-template-columns: repeat(2, 1fr);
		grid-column-gap: 20px;
		flex-wrap: wrap;
	}
	.root .list-svg{
		display: flex;
		justify-content: space-between;
		column-gap: 20px;
		padding: 20px 0;
		border-bottom: solid 1px grey;
	}
	.root .list-svg .title{
		font-weight: 700;
		display: flex;
		column-gap: 20px;
		align-items: center;
	}
	@media(max-width: 1000px){
		.root .list{
			grid-template-columns: 1fr;
		}
	}
</style>