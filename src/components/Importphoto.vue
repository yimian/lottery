<template>
  <el-dialog
    :visible="visible"
    :append-to-body="true"
    width="320px"
    @close="$emit('update:visible', false)"
    class="c-Importphoto"
  >
    <el-row>
      <label for="idinput">抽奖号码</label>
      <el-input
        id="idinput"
        size="mini"
        type="number"
        v-model="id"
        :min="0"
        :max="config.number"
      ></el-input>
    </el-row>
    <el-row>
      <label for="nameinput">人员姓名</label>
      <el-input id="nameinput" v-model="name" size="mini"></el-input>
    </el-row>
    <el-row>
      <label for="idinput">照片选择</label>
      <span class="selectbg" :data-tip="filename">
        <input
          ref="uploadinput"
          class="upload"
          type="file"
          accept=".jpg,.png"
          @change="inputChange"
        />
      </span>
    </el-row>
    <el-row class="photo">
      <label>已选照片</label>
      <img v-if="value" :src="value" alt="img" :width="140" :height="140" />
      <span v-else>暂未选择</span>
    </el-row>
    <el-row>
      支持 jpg 和 png，大小不能超过150kb，建议 20-50kb，建议尺寸 160 * 160px
    </el-row>
    <el-row class="center">
      <el-button size="mini" @click="$emit('update:visible', false)">
        取消
      </el-button>
      <el-button size="mini" type="primary" @click="saveHandler">
        保存
      </el-button>
    </el-row>
  </el-dialog>
</template>
<script>
import { database, DB_STORE_NAME } from '@/helper/db';

export default {
  name: 'Importphoto',
  props: {
    visible: Boolean
  },
  computed: {
    config: {
      get() {
        return this.$store.state.config;
      }
    }
  },
  data() {
    return {
      id: 0,
      name: '',
      value: '',
      filename: '点击选择照片'
    };
  },
  methods: {
    inputChange(e) {
      const fileList = e.target.files;
      const formData = new FormData();
      formData.append('uploadImg', fileList[0]);
      const reader = new FileReader();
      const AllowImgFileSize = 150; // KB
      const file = fileList[0];
      const fileSize = file.size / 1024;
      if (file) {
        this.filename = file.name;
        reader.readAsDataURL(file);
        reader.onload = () => {
          if (AllowImgFileSize && AllowImgFileSize < fileSize) {
            return this.$message.error('不允许上传大于 150KB 的图片');
          } else {
            this.value = reader.result;
          }
        };
      }
    },
    async saveHandler() {
      const { id, name, value } = this;
      const ID = Number(id);
      if (!ID || ID <= 0) {
        return this.$message.error('号码须为大于 0 的整数');
      }
      if (!value) {
        return this.$message.error('请选择照片');
      }
      const Data = await database.get(DB_STORE_NAME, ID);
      const param = {
        id: ID,
        name,
        value
      };
      database[Data ? 'edit' : 'add'](
        DB_STORE_NAME,
        Data ? ID : param,
        Data ? param : null
      )
        .then(res => {
          if (res) {
            this.$refs.uploadinput.value = '';
            this.value = '';
            this.filename = '点击选择照片';
            this.$emit('update:visible', false);
            this.$emit('getPhoto');
            this.$message({
              message: '保存成功',
              type: 'success'
            });
          } else {
            this.$message.error('保存失败');
          }
        })
        .catch(err => {
          this.$message.error(err.message);
        });
    }
  }
};
</script>
<style lang="scss">
.c-Importphoto {
  .el-dialog {
    height: 420px;
  }
  label {
    margin-right: 20px;
    vertical-align: top;
  }
  .el-input {
    width: 140px;
  }
  .el-row {
    padding: 5px 0;
  }
  .center {
    text-align: center;
  }
  .selectbg {
    display: inline-block;
    border: 1px solid #ccc;
    border-radius: 2px;
    width: 140px;
    height: 28px;
    position: relative;
    box-sizing: border-box;
    &::before {
      content: attr(data-tip);
      width: 100%;
      height: 100%;
      text-align: center;
      position: absolute;
      top: 0;
      left: 0;
      line-height: 28px;
      cursor: pointer;
      overflow: hidden;
      font-size: 12px;
    }
    .upload {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      opacity: 0;
      z-index: 10;
      cursor: pointer;
    }
  }
  .photo {
    img {
      border: 1px solid #ccc;
    }
    span {
      display: inline-block;
      border: 1px solid #ccc;
      border-radius: 2px;
      width: 140px;
      height: 140px;
      line-height: 140px;
      text-align: center;
      position: relative;
      box-sizing: border-box;
    }
  }
}
</style>
