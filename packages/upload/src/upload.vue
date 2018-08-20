<script>
import ajax from './ajax';
import UploadDragger from './upload-dragger.vue';

export default {
  inject: ['uploader'],
  components: {
    UploadDragger
  },
  props: {
    type: String,
    action: {
      type: String,
      required: true
    },
    name: {
      type: String,
      default: 'file'
    },
    data: Object,
    headers: Object,
    withCredentials: Boolean,
    multiple: Boolean,
    accept: String,
    onStart: Function,
    onProgress: Function,
    onSuccess: Function,
    onError: Function,
    beforeUpload: Function,
    drag: Boolean,
    onPreview: {
      type: Function,
      default: function() {}
    },
    onRemove: {
      type: Function,
      default: function() {}
    },
    fileList: Array,
    autoUpload: Boolean,
    listType: String,
    httpRequest: {
      type: Function,
      default: ajax
    },
    disabled: Boolean,
    limit: Number,
    onExceed: Function
  },

  data() {
    return {
      mouseover: false,
      reqs: {}
    };
  },

  methods: {
    isImage(str) {
      return str.indexOf('image') !== -1;
    },
    handleChange(ev) {
      const files = ev.target.files;

      if (!files) return;
      this.uploadFiles(files);
    },
    uploadFiles(files) {
      if (this.limit && this.fileList.length + files.length > this.limit) {
        this.onExceed && this.onExceed(files, this.fileList);
        return;
      }

      let postFiles = Array.prototype.slice.call(files);
      if (!this.multiple) { postFiles = postFiles.slice(0, 1); }

      if (postFiles.length === 0) { return; }

      postFiles.forEach(rawFile => {
        this.onStart(rawFile);
        if (this.autoUpload) this.upload(rawFile);
      });
    },
    upload(rawFile) {
      this.$refs.input.value = null;

      function splice() {
        var chunkSize = 2 * 1024 * 1024;
        let fileSize = rawFile.size;
        let quence = [];
        //if(rawFile.size > chunkSize) {//文件大于阀值，进行切块
        //切块发送
        var chunks = Math.ceil(fileSize / chunkSize); //分割块数
        for (var i = 0; i < chunks; i++) {
          var startIdx = i * chunkSize; //块的起始位置
          var endIdx = startIdx + chunkSize; //块的结束位置
          if (endIdx > fileSize) {
            endIdx = fileSize;
          }
          var lastChunk = false;
          if (i == (chunks - 1)) {
            lastChunk = true;
          }
          var file = {
            status: 'ready',
            name: rawFile.name,
            size: rawFile.size,
            type: rawFile.type,
            percentage: 0,
            uid: rawFile.uid,
            startIdx: startIdx,
            endIdx: endIdx,
            chunked: true,
            currChunk: i,
            totalChunk: chunks,
            file: rawFile.slice(startIdx, endIdx)
          };
          _this2.post(file);
        }
      }
      if (!this.beforeUpload) {
        return this.post(rawFile);
      }

      const before = this.beforeUpload(rawFile);
      if (before && before.then) {
        before.then(processedFile => {
          const fileType = Object.prototype.toString.call(processedFile);

          if (fileType === '[object File]' || fileType === '[object Blob]') {
            if (fileType === '[object Blob]') {
              processedFile = new File([processedFile], rawFile.name, {
                type: rawFile.type
              });
            }
            for (const p in rawFile) {
              if (rawFile.hasOwnProperty(p)) {
                processedFile[p] = rawFile[p];
              }
            }
            this.post(processedFile);
          } else {
            splice();
          }
        }, () => {
          this.onRemove(null, rawFile);
        });
      } else if (before !== false) {
        splice();
      } else {
        this.onRemove(null, rawFile);
      }
    },
    abort(file) {
      const { reqs } = this;
      if (file) {
        let uid = file;
        if (file.uid) uid = file.uid;
        if (reqs[uid]) {
          reqs[uid].abort();
        }
      } else {
        Object.keys(reqs).forEach((uid) => {
          if (reqs[uid]) reqs[uid].abort();
          delete reqs[uid];
        });
      }
    },
    post: function post(rawFile) {

      let data = this.data;
      let file = rawFile;
      if (!!rawFile.chunked) {
        data = Object.assign({}, { currChunk: rawFile.currChunk, totalChunk: rawFile.totalChunk, filename: rawFile.name, type: rawFile.type }, this.data);
        file = new Blob([rawFile.file], { type: rawFile.type });
      }
      var _this3 = this;
      var uid = rawFile.uid;
      var options = {
        headers: this.headers,
        withCredentials: this.withCredentials,
        file: file,
        data: data,
        filename: this.name,
        action: this.action,
        onProgress: function onProgress(e) {
          _this3.onProgress(e, rawFile);
        },
        onSuccess: function onSuccess(res) {
          if (res.stateCode == 200) {
            _this3.onSuccess(res, rawFile);
            delete _this3.reqs[uid];
          }
        },
        onError: function onError(err) {
          _this3.onError(err, rawFile);
          delete _this3.reqs[uid];
        }
      };
      var req = this.httpRequest(options);
      this.reqs[uid] = req;
      if (req && req.then) {
        req.then(options.onSuccess, options.onError);
      }
    },
    handleClick() {
      if (!this.disabled) {
        this.$refs.input.value = null;
        this.$refs.input.click();
      }
    },
    handleKeydown(e) {
      if (e.target !== e.currentTarget) return;
      if (e.keyCode === 13 || e.keyCode === 32) {
        this.handleClick();
      }
    }
  },

  render(h) {
    let {
      handleClick,
      drag,
      name,
      handleChange,
      multiple,
      accept,
      listType,
      uploadFiles,
      disabled,
      handleKeydown
    } = this;
    const data = {
      class: {
        'el-upload': true
      },
      on: {
        click: handleClick,
        keydown: handleKeydown
      }
    };
    data.class[`el-upload--${listType}`] = true;
    return ( <
      div { ...data } tabindex = "0" > {
        drag ?
        < upload - dragger disabled = { disabled } on - file = { uploadFiles } > { this.$slots.default } < /upload-dragger> :
          this.$slots.default
      } <
      input class = "el-upload__input"
      type = "file"
      ref = "input"
      name = { name } on - change = { handleChange } multiple = { multiple } accept = { accept } > < /input> <
      /div>
    );
  }
};
</script>