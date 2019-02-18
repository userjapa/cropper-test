<template>
  <div class="cropper">
    <div class="cropper__form">
      <div class="cropper__form__input">
        <input ref="text" type="text" :value="file ? file.name : 'Upload Image'" readonly />
        <input ref="file" type="file" @change="changedFile($event.target.files[0])" style="display: none;" />
        <button type="button" @click="$refs['file'].click()">Upload</button>
      </div>
      <div class="cropper__form__message">
        <span v-show="error">{{ error }}</span>
      </div>
    </div>
    <div class="cropper__view">
      <img :src="source">
    </div>
    <div class="cropper__modal" v-show="showModal">
      <div class="cropper__modal__box">
        <div class="cropper__modal__box__images">
          <img ref="img" :src="source">
        </div>
        <div class="cropper__modal__box__options">
          <div class="cropper__modal__box__options__inputs">
            <div class="cropper__modal__box__options__inputs__item">
              <label for="cropper-width">Width</label>
              <input id="cropper-width" type="number" name="width" v-model.number="size.width" step="1" min="0" :max="img.width" />
            </div>
            <div class="cropper__modal__box__options__inputs__item">
              <label for="cropper-height">Height</label>
              <input id="cropper-height" type="number" name="heigth" v-model.number="size.height" step="1" min="0" :max="img.height" />
            </div>
          </div>
          <div class="cropper__modal__box__options__btn">
            <button type="button" name="crop" @click="crop()">Crop</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Cropper from 'cropperjs'

export default {
  name: 'app',
  data () {
    return {
      file: null,
      source: '',
      cropper: null,
      showModal: false,
      error: '',
      size: {
        width: 0,
        height: 0
      },
      img: {
        width: 0,
        height: 0
      },
      changed: false,
      cropping: false
    }
  },
  props: {
    onchanged: {
      type: Function,
      default: () => console.log('Changed callback')
    }
  },
  methods: {
    delay (time) {
      return new Promise(resolve => setTimeout(resolve, time))
    },
    async changedFile (file) {
      if (!file) {
        if (this.file) {
          this.file = null
          this.source = ''
          this.cropper = null
        }
      } else {
        this.error = ''
        try {
          if (/(jpeg|jpg|png|gif|svg)/g.test(file.type.split('/').pop().toLowerCase())) {
            this.file = file
            this.source = await this.readFile(file)
            await this.delay(500)
            this.setCropper(this.$refs['img'])
            this.showModal = true
          } else {
            throw Error('File format not acceptable.')
          }
        } catch (e) {
          this.error = 'File format not acceptable.'
        }
      }
    },
    readFile (file) {
      return new Promise((resolve, reject) => {
        let reader =  new FileReader()
        reader.onload = () => {
          resolve(reader.result)
        }
        reader.onerror = () => {
          reject(reader.error)
        }
        reader.readAsDataURL(file)
      })
    },
    setCropper (image) {
      let self = this
      new Cropper(image, {
        viewMode: 2,
        dragMode: 'move',
        responsive: true,
        modal: true,
        center: true,
        background: true,
        scalable: true,
        crop () {
          if (self.changed) self.changed = false
          else {
            self.cropping = true
            const data = this.cropper.getCropBoxData()
            console.log('cropped')
            self.size.left = data.left
            self.size.top = data.top
            self.size.width = data.width
            self.size.height = data.height
          }
        },
        ready () {
          self.cropper = this.cropper
          self.size = this.cropper.getCropBoxData()
          self.img = this.cropper.getImageData()
          this.cropper.setCropBoxData(self.img)
          self.size.top = self.img.top
          self.size.left = self.img.left
          self.size.width = self.img.width
          self.size.height = self.img.height
        }
      })
    },
    crop () {
      this.cropper.getCroppedCanvas().toBlob(async blob => {
        const dateObj = new Date()
        blob.lastModified = dateObj.getTime()
        blob.lastModifiedDate = dateObj
        blob.name = this.file.name
        this.file = blob
        this.onchanged(blob)
        this.source = await this.readFile(blob)
        this.showModal = false
        this.cropper.destroy()
        this.cropper = null
        this.$refs['file'].value = ''
      })
    }
  },
  watch: {
    async 'size.width' (value) {
      if (this.cropping) this.cropping = false
      else {
        this.changed = true
        if (typeof value == 'string') this.size.width = 0
        else this.cropper.setCropBoxData(this.size)
      }
    },
    async 'size.height' (value) {
      if (this.cropping) this.cropping = false
      else {
        this.changed = true
        if (typeof value == 'string') this.size.height = 0
        else this.cropper.setCropBoxData(this.size)
      }
    }
  }
}
</script>

<style lang="scss" scoped>
@import './../node_modules/cropperjs/dist/cropper.css';

.cropper {
  position: relative;
  &__form {
    &__input {
      input[type="text"] {
        cursor: not-allowed;
        padding: 5.5px 10px;
        border: 2px solid #BBB;
        border-radius: 3px;
        width: auto;
        background-color: #EEE;
      }
      button {
        cursor: pointer;
        padding: 7.5px 12px;
        border: none;
        border-radius: 3px;
      }
    }
  }
  &__modal {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background-color: rgba(#000, .75);
    z-index: 99999;
    max-width: 100%;
    max-height: 100%;
    &__box {
      position: relative;
      padding: 20px 50px;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      &__images {
        flex-grow: 1;
        margin-bottom: 10px;
        img {
          max-width: calc(100vw - 100px);
          max-height: calc(60vh);
        }
      }
      &__options {
        flex-grow: 1;
        display: flex;
        justify-content: center;
        align-items: center;
        flex-direction: column;
        &__inputs {
          display: flex;
          justify-content: center;
          align-items: center;
          &__item {
            display: flex;
            justify-content: center;
            align-items: center;
            flex-direction: column;
            color: #FFF;
            margin-bottom: 10px;
            font-size: 18px;
            input {
              padding: 5px 10px;
              margin: 0 5px;
              border: none;
              border-radius: 3px;
              width: 90px;
              font-size: 16px;
            }
          }
        }
        &__btn {
          button {
            cursor: pointer;
            padding: 8px 12px;
            border: none;
            border-radius: 3px;
            width: 230px;
            font-size: 18px;
          }
        }
      }
    }
  }
}
</style>
