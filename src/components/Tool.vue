<template>
  <div>
    <el-dropdown :disabled="running" @command="handleCommand">
      <el-button
        :disabled="running"
        type="text"
        class="el-button__white"
        icon="el-icon-s-tools"
        circle
      ></el-button>
      <el-dropdown-menu slot="dropdown">
        <el-dropdown-item command="system">
          系统配置
        </el-dropdown-item>
        <el-dropdown-item command="config">
          奖项配置
        </el-dropdown-item>
        <el-dropdown-item command="reset">
          重置配置
        </el-dropdown-item>
        <el-dropdown-item command="import">导入名单</el-dropdown-item>
        <el-dropdown-item command="importPhoto">导入照片</el-dropdown-item>
        <el-dropdown-item command="bulkImportPhoto">批量导入</el-dropdown-item>
      </el-dropdown-menu>
    </el-dropdown>
    <!-- <el-button @click="startHandler" type="primary" size="mini">{{
      running ? '停止' : '开始'
    }}</el-button> -->

    <el-dialog
      :append-to-body="true"
      :visible.sync="showSetwat"
      class="setwat-dialog"
      width="460px"
    >
      <el-form ref="form" :model="form" label-width="80px" size="mini">
        <el-form-item label="抽取奖项">
          <el-select v-model="form.category" placeholder="请选取本次抽取的奖项">
            <el-option
              v-for="(item, index) in categorys"
              :key="index"
              :label="item.label"
              :value="item.value"
            ></el-option>
          </el-select>
        </el-form-item>

        <el-form-item label=" " v-if="form.category">
          <span>
            共&nbsp;
            <span class="colorred">{{ config[form.category] }}</span>
            &nbsp;名
          </span>
          <span :style="{ marginLeft: '20px' }">
            剩余&nbsp;
            <span class="colorred">{{ remain }}</span>
            &nbsp;名
          </span>
        </el-form-item>
        <el-form-item>
          <el-image
            style="width: 200px; height: 200px;"
            v-if="currentPrize"
            :src="currentPrize"
          />
        </el-form-item>
        <el-form-item label="抽取方式">
          <el-select v-model="form.mode" placeholder="请选取本次抽取方式">
            <el-option label="抽 1 人" :value="1"></el-option>
            <el-option label="抽 5 人" :value="5"></el-option>
            <el-option label="一次抽取完" :value="0"></el-option>
            <el-option label="自定义" :value="99"></el-option>
          </el-select>
        </el-form-item>

        <el-form-item label="抽取人数" v-if="form.mode === 99">
          <el-input
            v-model="form.qty"
            type="number"
            :clearable="true"
            :min="1"
            :max="remain ? remain : 100"
            :step="1"
          ></el-input>
        </el-form-item>

        <el-form-item label="全员参与">
          <el-switch v-model="form.allin"></el-switch>
          <span :style="{ fontSize: '12px' }">
            (开启后将在全体成员[无论之前有无中奖]中抽奖)
          </span>
        </el-form-item>

        <el-form-item
          style="margin-top: 10px; display: flex; justify-content: flex-end;"
        >
          <el-button @click="showSetwat = false">取消</el-button>
          <el-button type="primary" @click="onSubmit">立即抽奖</el-button>
        </el-form-item>
      </el-form>
    </el-dialog>

    <el-dialog
      :append-to-body="true"
      :visible.sync="showImport"
      class="import-dialog"
      width="400px"
    >
      <el-input
        type="textarea"
        v-model="listStr"
        :rows="10"
        placeholder="请输入对应的号码和名单(可直接从 excel 复制)，格式(号码 名字)，导入的名单将代替号码显示在抽奖中。如：
1 张三
2 李四
3 王五
				"
      ></el-input>
      <div class="footer">
        <el-button size="mini" @click="showImport = false">取消</el-button>
        <el-button size="mini" type="primary" @click="transformList"
          >确定</el-button
        >
      </div>
    </el-dialog>
    <Importphoto
      :visible.sync="showImportphoto"
      @getPhoto="$emit('getPhoto')"
    ></Importphoto>
    <BulkImportphoto
      :visible.sync="showBulkImportphoto"
      @getPhoto="$emit('getPhoto')"
    ></BulkImportphoto>

    <el-dialog
      :visible.sync="showRemoveoptions"
      width="300px"
      class="c-removeoptions"
      :append-to-body="true"
    >
      <el-form ref="form" :model="removeInfo" label-width="80px" size="mini">
        <el-form-item label="重置选项">
          <el-radio-group v-model="removeInfo.type">
            <el-radio border :label="2">重置名单</el-radio>
            <el-radio border :label="3">重置照片</el-radio>
            <el-radio border :label="4">重置抽奖结果</el-radio>
            <el-radio border :label="1">重置抽奖配置</el-radio>
            <el-radio border :label="5">重置系统配置</el-radio>
            <el-radio border :label="0">重置全部数据</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item>
          <el-button @click="showRemoveoptions = false">取消</el-button>
          <el-button type="primary" @click="resetConfig">确定重置</el-button>
        </el-form-item>
      </el-form>
    </el-dialog>
  </div>
</template>

<script>
import {
  clearData,
  removeData,
  systemConfigField,
  configField,
  listField,
  resultField,
  conversionCategoryName
} from '@/helper/index';
import Importphoto from './Importphoto';
import BulkImportphoto from './BulkImportphoto';
import { database, DB_STORE_NAME } from '@/helper/db';

export default {
  components: { Importphoto, BulkImportphoto },
  props: {
    running: Boolean,
    closeRes: Function
  },
  data() {
    return {
      showSetwat: false,
      showImport: false,
      showImportphoto: false,
      showBulkImportphoto: false,
      showRemoveoptions: false,
      removeInfo: { type: 2 },
      form: {
        category: '',
        mode: 1,
        qty: 1,
        allin: false
      },
      listStr: ''
    };
  },
  computed: {
    config() {
      return this.$store.state.config;
    },
    photos() {
      return this.$store.state.photos;
    },
    currentPrize() {
      if (!this.form.category) {
        return '';
      }

      const item = this.photos.find(i => i.id === this.form.category);
      return item ? item.value : '';
    },
    remain() {
      if (!this.config[this.form.category]) {
        return 0;
      }

      const count =
        this.config[this.form.category] -
        (this.result[this.form.category]
          ? this.result[this.form.category].length
          : 0);
      return count < 0 ? 0 : count;
    },
    result() {
      return this.$store.state.result;
    },
    categorys() {
      const options = [];
      Object.keys(this.config).forEach(key => {
        const item = this.config[key];
        if (Number(item) > 0) {
          let name = conversionCategoryName(key, this.config);
          if (name) {
            options.push({
              label: name,
              value: key
            });
          }
        }
      });
      return options;
    }
  },
  watch: {
    showRemoveoptions(v) {
      if (!v) {
        this.removeInfo.type = null;
      }
    }
  },
  methods: {
    handleCommand(command) {
      if (command === 'system') {
        this.$emit('show-sys-config');
      } else if (command === 'config') {
        this.$emit('show-config');
      } else if (command === 'reset') {
        this.showRemoveoptions = true;
      } else if (command === 'import') {
        this.showImport = true;
      } else if (command === 'importPhoto') {
        this.showImportphoto = true;
      } else if (command === 'bulkImportPhoto') {
        this.showBulkImportphoto = true;
      }
    },
    resetConfig() {
      const type = this.removeInfo.type;
      if (type === undefined || type === null) {
        return;
      }

      this.$confirm('此操作将重置所选数据，是否继续?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          switch (type) {
            case 0:
              clearData();
              this.$store.commit('setClearStore');
              this.form.category = '';
              this.$emit('reset-category');
              database.clear(DB_STORE_NAME);
              break;
            case 1:
              removeData(configField);
              this.$store.commit('setClearConfig');
              this.form.category = '';
              this.$emit('reset-category');
              break;
            case 2:
              removeData(listField);
              this.$store.commit('setClearList');
              break;
            case 3:
              database.clear(DB_STORE_NAME);
              this.$store.commit('setClearPhotos');
              break;
            case 4:
              removeData(resultField);
              this.$store.commit('setClearResult');
              break;
            case 5:
              removeData(systemConfigField);
              this.$store.commit('setClearSysConfig');
              break;
            default:
              break;
          }

          this.closeRes && this.closeRes();
          this.form.category = '';
          this.showRemoveoptions = false;
          if ([0, 1, 4].includes(type)) {
            this.$store.commit('resetOldPrizeKeys');
          }

          this.$message({
            type: 'success',
            message: '重置成功!'
          });

          this.$nextTick(() => {
            this.$emit('resetConfig');
          });
        })
        .catch(() => {
          this.$message({
            type: 'info',
            message: '已取消'
          });
        });
    },
    onSubmit() {
      if (!this.form.category) {
        return this.$message.error('请选择本次抽取的奖项');
      }
      if (this.remain <= 0) {
        return this.$message.error('该奖项剩余人数不足');
      }
      if (this.form.mode === 99) {
        if (this.form.qty <= 0) {
          return this.$message.error('必须输入本次抽取人数');
        }
        if (this.form.qty > this.remain) {
          return this.$message.error('本次抽奖人数已超过本奖项的剩余人数');
        }
      }
      if (this.form.mode === 1 || this.form.mode === 5) {
        if (this.form.mode > this.remain) {
          return this.$message.error('本次抽奖人数已超过本奖项的剩余人数');
        }
      }
      this.showSetwat = false;
      this.$emit(
        'toggle',
        Object.assign({}, this.form, { remain: this.remain })
      );
    },
    startHandler() {
      // this.$emit('toggle');
      if (!this.running) {
        this.showSetwat = true;
      }
    },
    transformList() {
      const { listStr } = this;
      if (!listStr) {
        this.$message.error('没有数据');
      }
      const list = [];
      const rows = listStr.split('\n');
      if (rows && rows.length) {
        rows.forEach(item => {
          const rowList = item.split(/\t|\s/);
          if (rowList.length >= 2) {
            const key = Number(rowList[0].trim());
            const name = rowList[1].trim();
            key &&
              list.push({
                key,
                name
              });
          }
        });
      }
      this.$store.commit('setList', list);
      this.$message({
        message: '保存成功',
        type: 'success'
      });
      this.showImport = false;
      this.$nextTick(() => {
        this.$emit('resetConfig');
      });
    }
  }
};
</script>
<style lang="scss">
#tool {
  position: fixed;
  width: 60px;
  top: 50%;
  right: 20px;
  transform: translateY(-50%);
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  .el-button + .el-button {
    margin-top: 20px;
    margin-left: 0px;
  }
}
.setwat-dialog {
  .colorred {
    color: red;
    font-weight: bold;
  }
}
.import-dialog {
  .footer {
    height: 50px;
    line-height: 50px;
    text-align: center;
  }
}
.c-removeoptions {
  .el-dialog {
    height: 320px;
  }
  .el-radio.is-bordered + .el-radio.is-bordered {
    margin-left: 0px;
  }
  .el-radio.is-bordered {
    margin-bottom: 10px;
  }
}
</style>
