<template>
  <div v-loading="loading"
       element-loading-text="加载中..."
       element-loading-spinner="iconfont iconfont-loading icondocker"
       class="setting-registry-container">
    <!--registry-create-dialog-->
    <el-dialog :title="mode === 'create'?'添加':'编辑'"
               :visible.sync="dialogRegistryFormVisible"
               :close-on-click-modal="false"
               custom-class="dialog-style"
               width="35%">
      <el-form ref="registry"
               :rules="rules"
               label-width="150px"
               label-position="left"
               :model="registry">
        <el-form-item label="默认使用"
                      prop="is_default">
          <el-checkbox v-model="registry.is_default"></el-checkbox>
        </el-form-item>
        <el-form-item label="提供商"
                      prop="reg_provider">
          <el-select v-model="registry.reg_provider"
                     @change="changeProvider"
                     style="width: 100%;"
                     size="small"
                     placeholder="请选择镜像仓库提供商">
            <el-option v-for="(provider,index) in providers" :key="index" :value="provider.value"
                       :label="provider.label">
              <i :class="provider.icon"></i> <span>{{provider.label}}</span>
            </el-option>
          </el-select>
        </el-form-item>
        <el-form-item v-if="registry.reg_provider === 'swr' || registry.reg_provider === 'ecr'"
                      :label="providerMap[registry.reg_provider].region"
                      prop="region">
          <el-input size="small"
                    clearable
                    v-model="registry.region"></el-input>
        </el-form-item>
        <el-form-item v-if="registry.reg_provider"
                      :label="providerMap[registry.reg_provider].reg_addr"
                      prop="reg_addr">
          <el-input size="small"
                    clearable
                    v-model="registry.reg_addr"></el-input>
        </el-form-item>
        <el-form-item v-if="registry.reg_provider && registry.reg_provider !=='ecr'" prop="namespace">
          <template slot="label">
            {{ registry.reg_provider ? providerMap[registry.reg_provider].namespace : 'Namespace' }}
          </template>
          <el-input size="small"
                    clearable
                    v-model="registry.namespace"></el-input>
        </el-form-item>
        <el-form-item v-if="registry.reg_provider" :label="providerMap[registry.reg_provider].access_key"
                      :rules="{ required: true, message: `请输入 ${providerMap[registry.reg_provider].access_key}`, trigger: ['blur'] }"
                      prop="access_key">
          <el-input size="small"
                    clearable
                    v-model="registry.access_key"></el-input>
        </el-form-item>
        <el-form-item v-if="registry.reg_provider" :label="providerMap[registry.reg_provider].secret_key"
                      :rules="{ required: true, message: `请输入 ${providerMap[registry.reg_provider].secret_key}`, trigger: ['blur'] }"
                      prop="secret_key">
          <el-input size="small"
                    clearable
                    type="passsword"
                    v-model="registry.secret_key"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer"
           class="dialog-footer">
        <el-button size="small"
                   @click="dialogRegistryFormVisible = false">取 消</el-button>
        <el-button :plain="true"
                   type="success"
                   size="small"
                   @click="mode === 'create' ? registryAction('add'): registryAction('update')">保存</el-button>
      </div>
    </el-dialog>
    <!--registry-create-dialog-->
    <div class="section">
      <el-alert type="info"
                :closable="false">
        <template>
          支持集成云厂商提供的标准镜像仓库或者自建镜像仓库<br />
          镜像仓库主要用于存放构建输出的镜像以及预发布的镜像，涉及到工作流的运行，以及集成环境的启动<br />
          集成镜像仓库可参考
          <el-link style="font-size: 14px; vertical-align: baseline;"
                   type="primary"
                   :href="`https://docs.koderover.com/zadig/settings/image-registry/`"
                   :underline="false"
                   target="_blank">帮助文档</el-link>
        </template>
      </el-alert>
      <div class="sync-container">
        <el-button :plain="true"
                   @click="addRegistryBtn"
                   size="small"
                   type="success">新建</el-button>
      </div>
      <div class="registry-list">
        <template>
          <el-table :data="allRegistry"
                    style="width: 100%;">
            <el-table-column label="提供商/地址/Namespace">
              <template slot-scope="scope">
                <i :class="getProviderMap(scope.row.reg_provider,'icon')"></i>
                <span v-if="scope.row.namespace">{{`${scope.row.reg_addr.split('://')[1]}/${scope.row.namespace}`}}</span>
                <span v-else>{{`${scope.row.reg_addr.split('://')[1]}`}}</span>
              </template>
            </el-table-column>
            <el-table-column width="150px"
                             label="默认使用">
              <template slot-scope="scope">
                <el-tag size="small"
                        v-if="scope.row.is_default">默认使用</el-tag>
                <span v-else>-</span>
              </template>
            </el-table-column>
            <el-table-column width="220px"
                             label="修改时间">
              <template slot-scope="scope">
                <i class="el-icon-time"></i>
                <span
                      style="margin-left: 5px;">{{ $utils.convertTimestamp(scope.row.update_time) }}</span>
              </template>
            </el-table-column>
            <el-table-column label="最后修改">
              <template slot-scope="scope">
                <span>{{ scope.row.update_by}}</span>
              </template>
            </el-table-column>

            <el-table-column label="操作"
                             width="180px">
              <template slot-scope="scope">
                <el-button @click="registryAction('edit',scope.row)"
                           size="mini">编辑</el-button>
                <el-button @click="registryAction('delete',scope.row)"
                           size="mini"
                           type="danger">删除</el-button>
              </template>
            </el-table-column>
          </el-table>
        </template>
      </div>
    </div>
  </div>
</template>

<script>

import { getRegistryListAPI, createRegistryAPI, updateRegistryAPI, deleteRegistryAPI } from '@api'
import { keyBy, cloneDeep } from 'lodash'
import bus from '@utils/event_bus'
export default {
  data () {
    return {
      allRegistry: [],
      mode: '',
      registry: {
        namespace: '',
        reg_addr: '',
        access_key: '',
        secret_key: '',
        reg_provider: '',
        region: '',
        is_default: false
      },
      providers: [
        {
          value: 'acr',
          label: '阿里云 ACR',
          reg_addr: '地址',
          namespace: '命名空间',
          access_key: 'Docker 用户名',
          secret_key: 'Docker 密码',
          icon: 'iconfont logo iconaliyun'
        },
        {
          value: 'swr',
          label: '华为云 SWR',
          reg_addr: '地址',
          namespace: '组织名称',
          region: '区域',
          access_key: 'Access Key',
          secret_key: 'Secret Key',
          icon: 'iconfont logo iconhuawei'
        },

        {
          value: 'tcr',
          label: '腾讯云 TCR',
          reg_addr: '地址',
          namespace: '命名空间',
          access_key: 'Docker 用户名',
          secret_key: 'Docker 密码',
          icon: 'iconfont logo icontengxunyun'
        },
        {
          value: 'harbor',
          label: 'Harbor',
          reg_addr: '地址',
          namespace: '项目',
          access_key: 'Docker 用户名',
          secret_key: 'Docker 密码',
          icon: 'iconfont logo iconHarbor'
        },
        {
          value: 'dockerhub',
          label: 'DockerHub',
          reg_addr: '地址',
          namespace: 'Namespace',
          access_key: 'Docker 用户名',
          secret_key: 'Docker 密码',
          icon: 'iconfont logo icondocker'
        },
        {
          value: 'ecr',
          label: 'Amazon ECR',
          reg_addr: 'URI',
          namespace: '',
          region: 'Region',
          access_key: 'Access Key ID',
          secret_key: 'Secret Access Key',
          icon: 'iconfont logo iconaws'
        },
        {
          value: 'native',
          label: '其它',
          reg_addr: '地址',
          namespace: 'Namespace',
          access_key: 'Docker 用户名',
          secret_key: 'Docker 密码',
          icon: 'iconfont logo iconqita'
        }
      ],
      dialogRegistryFormVisible: false,
      loading: true,
      rules: {
        reg_provider: [{ required: true, message: '请选择镜像仓库提供商', trigger: ['blur'] }],
        reg_addr: [{
          required: true,
          message: '请输入 URL',
          trigger: ['blur']
        },
        {
          type: 'url',
          message: '请输入正确的 URL，包含协议',
          trigger: ['blur']
        }],
        region: [{ required: true, message: '请输入区域', trigger: ['blur'] }],
        namespace: [{ required: true, message: '请输入 Namespace', trigger: ['blur'] }]
      }
    }
  },
  computed: {
    providerMap () {
      return keyBy(this.providers, 'value')
    }
  },
  methods: {
    addRegistryBtn () {
      if (this.allRegistry.length === 0) {
        this.registry.is_default = true
      } else {
        this.registry.is_default = false
      }
      this.registry = {
        namespace: '',
        reg_addr: '',
        access_key: '',
        secret_key: '',
        reg_provider: '',
        region: '',
        is_default: false
      }
      this.mode = 'create'
      this.dialogRegistryFormVisible = true
    },
    getProviderMap (name, type) {
      const providerMap = this.providerMap
      if (providerMap[name] && type) {
        return providerMap[name][type]
      } else {
        return providerMap.native.icon
      }
    },
    changeProvider (val) {
      this.$nextTick(() => {
        this.registry = {
          namespace: '',
          reg_addr: val === 'dockerhub' ? 'https://registry.hub.docker.com' : '',
          access_key: '',
          secret_key: '',
          reg_provider: val,
          region: '',
          is_default: false
        }
        this.$refs.registry.clearValidate()
      })
    },
    registryAction (action, registry) {
      if (action === 'add') {
        this.$refs.registry.validate(valid => {
          if (valid) {
            const payload = this.registry
            this.dialogRegistryFormVisible = false
            this.addRegistry(payload)
          } else {
            return false
          }
        })
      } else if (action === 'edit') {
        this.registry = cloneDeep(registry)
        this.mode = 'edit'
        this.dialogRegistryFormVisible = true
        this.$nextTick(() => {
          this.$refs.registry.clearValidate()
        })
      } else if (action === 'update') {
        this.$refs.registry.validate(valid => {
          if (valid) {
            const id = this.registry.id
            const payload = this.registry
            this.dialogRegistryFormVisible = false
            this.updateRegistry(id, payload)
          } else {
            return false
          }
        })
      } else if (action === 'delete') {
        const id = registry.id
        this.$confirm(`确定要删除 ${registry.namespace} ?`, '确认', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(({ value }) => {
          deleteRegistryAPI(id).then((res) => {
            this.getRegistry()
            this.$message({
              message: '删除成功',
              type: 'success'
            })
          })
        })
      }
    },
    addRegistry (payload) {
      createRegistryAPI(payload).then((res) => {
        this.$refs.registry.resetFields()
        this.getRegistry()
        this.$message({
          type: 'success',
          message: '新增成功'
        })
      })
    },
    updateRegistry (id, payload) {
      updateRegistryAPI(id, payload).then((res) => {
        this.getRegistry()
        this.$message({
          type: 'success',
          message: '更新成功'
        })
      })
    },
    getRegistry () {
      this.loading = true
      getRegistryListAPI().then((res) => {
        this.loading = false
        this.allRegistry = res
      })
    }
  },
  created () {
    bus.$emit(`set-topbar-title`, { title: '镜像仓库', breadcrumb: [] })
    bus.$emit(`set-sub-sidebar-title`, {
      title: '',
      routerList: []
    })
    this.getRegistry()
  }
}
</script>

<style lang="less">
.setting-registry-container {
  position: relative;
  flex: 1;
  padding: 15px 30px;
  overflow: auto;
  font-size: 13px;

  .module-title h1 {
    margin-bottom: 1.5rem;
    font-weight: 200;
    font-size: 2rem;
  }

  .section {
    margin-bottom: 56px;

    .sync-container {
      padding-top: 15px;
      padding-bottom: 15px;
      overflow: hidden;

      .el-button--success.is-plain {
        color: #13ce66;
        background: #fff;
        border-color: #13ce66;
      }

      .el-button--success.is-plain:hover {
        color: #13ce66;
        background: #fff;
        border-color: #13ce66;
      }
    }

    .registry-list {
      padding-bottom: 30px;

      .logo {
        font-size: 20px;
        vertical-align: bottom;
      }
    }

    .dialog-style {
      .el-dialog__body {
        padding: 0 20px;
      }
    }
  }
}
</style>
