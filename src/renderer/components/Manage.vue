<template>
  <div>
    <div class="ui inverted dimmer" :class="[activeLoader && 'active']">
      <div class="ui text loader">{{ loaderText }}</div>
    </div>
    <div class="ui three item top attached tabular menu">
      <a class="item" data-tab="status-tab">
        <i class="dashboard icon"></i>
        状态
      </a>
      <a class="item active" data-tab="profiles-tab">
        <i class="send icon"></i>
        配置
      </a>
      <a class="item" data-tab="system-tab">
        <i class="linux icon"></i>
        系统
      </a>
    </div>

    <div class="ui bottom attached tab segment" data-tab="status-tab">
      <status-tab
        :routing="routing"
        :proxies="proxies"
        :mode="mode"
        @toggleRouting="toggleRouting"
      >
      </status-tab>
    </div>

    <div class="ui bottom attached tab segment active" data-tab="profiles-tab">
      <profiles-tab
        :profiles="profiles"
        :bus="bus"
      >
      </profiles-tab>
    </div>

    <div class="ui bottom attached tab segment" data-tab="system-tab">
      <system-tab
        :systemInfo="systemInfo"
        :vrouterInfo="vrouterInfo"
        :proxiesInfo="proxiesInfo"
      >
      </system-tab>
    </div>

    <profile-editor
      :editingClone="editingClone"
      :showProfileEditor="showProfileEditor"
      :bus="bus"
    >
    </profile-editor>

    <profile-importer
      :showProfileImporter="showProfileImporter"
      :bus="bus"
    >
    </profile-importer>

  </div>
</template>

<script>
/* global $ */
import Vue from 'vue'
import Utils from '@/lib/utils.js'
import VRouter from '@/lib/vrouter.js'
import VBox from '@/lib/vbox.js'
import StatusTab from './Manage/StatusTab.vue'
import ProfilesTab from './Manage/ProfilesTab.vue'
import SystemTab from './Manage/SystemTab'
import ProfileEditor from './Manage/ProfileEditor'
import ProfileImporter from './Manage/ProfileImporter'

const path = require('path')
const fs = require('fs-extra')
const { shell } = require('electron')

// let vueInstance = null
const appDir = Utils.getAppDir()
const vrouter = new VRouter(fs.readJsonSync(path.join(appDir, 'vrouter', 'config.json')))
Utils.configureLog(path.join(vrouter.cfgDirPath, vrouter.name + '.log'))
const bus = new Vue()

export default {
  name: 'manage',
  components: {
    StatusTab,
    ProfilesTab,
    SystemTab,
    ProfileEditor,
    ProfileImporter
  },
  data: function () {
    return {
      // 因为 profiles 是对象, 和 vrouter.config.profiles 指向同一个地址, 因此两者的值是同步的
      profiles: vrouter.config.profiles,
      activeLoader: false,
      loaderText: 'Loading',
      // vue 不能检测用常规方法对对象属性的进行'增加','删除'.
      // https://vuejs.org/v2/guide/list.html#Object-Change-Detection-Caveats
      editingClone: {
        index: 0,
        name: 'New Profile'
      },
      showProfileEditor: false,
      showProfileImporter: false,
      bus: bus,
      systemInfo: {
        // 为了和真实值一致, 这些值需要手动维护
        currentGWIP: '',
        currentDnsIP: ''
      },
      vrouterInfo: {
        // 这些值需要手动维护
        openwrtVersion: '',
        brLanIP: '',
        bridgeAdapter: '',
        lanIP: '',
        macAddress: '',
        ssVersion: '',
        ssrVersion: '',
        ktVersion: ''
      },
      proxiesInfo: {
        // 这些值需要手动维护
        enableTunnelDns: false,
        isTunnelDnsRunning: false,
        enableSs: false,
        isSsRunning: false,
        enableSsr: false,
        isSsrRunning: false,
        enableKt: false,
        isKtRunning: false
      }
    }
  },
  computed: {
    activedProfile: function () {
      let profile = {}
      this.profiles.forEach(p => {
        if (p.active) {
          profile = p
        }
      })
      return profile
    },
    proxies: function () {
      return Utils.getProxiesText(this.activedProfile.proxies)
    },
    mode: function () {
      if (this.activedProfile) {
        return Utils.getModeText(this.activedProfile.mode)
      }
    },
    routing: function () {
      return (this.systemInfo.currentGWIP === vrouter.ip) && (this.systemInfo.currentDnsIP === vrouter.ip)
    }
  },
  methods: {
    toggleRouting: async function () {
      this.activeLoader = true
      if (await this.routing) {
        await Utils.resetRoute()
      } else {
        await Utils.changeRouteTo(vrouter.ip)
      }
      await this.getSystemInfo()
      this.activeLoader = false
    },
    getSystemInfo: async function () {
      this.systemInfo.currentGWIP = await Utils.getCurrentGateway()
      this.systemInfo.currentDnsIP = await Utils.getCurrentDns()
    },
    getVrouterInfo: async function () {
      // can not make vrouterinfo an asyncComputed attribute for unkown error
      this.vrouterInfo.openwrtVersion = await vrouter.getOpenwrtVersion()
      this.vrouterInfo.brLanIP = await vrouter.getLan()
      this.vrouterInfo.bridgeAdapter = await VBox.getAssignedBridgeService(vrouter.name)
      this.vrouterInfo.lanIP = await vrouter.getWan()
      this.vrouterInfo.macAddress = await vrouter.getMacAddress()
      this.vrouterInfo.ssVersion = await vrouter.getSsVersion('shadowsocks', vrouter.config.proxiesInfo)
      this.vrouterInfo.ssrVersion = await vrouter.getSsVersion('shadowsocksr', vrouter.config.proxiesInfo)
      this.vrouterInfo.ktVersion = await vrouter.getKtVersion(vrouter.config.proxiesInfo)
    },
    getProxiesInfo: async function () {
      const proxiesInfo = vrouter.config.proxiesInfo

      this.proxiesInfo.enableTunnelDns = this.activedProfile.enableTunnelDns
      this.proxiesInfo.isTunnelDnsRunning = await vrouter.isTunnelDnsRunning(this.activedProfile.proxies, proxiesInfo)

      this.proxiesInfo.enableSs = /^(ss|ssKt)$/ig.test(this.activedProfile.proxies)
      this.proxiesInfo.isSsRunning = await vrouter.isSsRunning(this.activedProfile.proxies, proxiesInfo)
      this.proxiesInfo.enableSsr = /ssr/ig.test(this.activedProfile.proxies)
      this.proxiesInfo.isSsrRunning = await vrouter.isSsRunning(this.activedProfile.proxies, proxiesInfo)

      this.proxiesInfo.enableKt = /kt/ig.test(this.activedProfile.proxies)
      this.proxiesInfo.isKtRunning = await vrouter.isKtRunning(proxiesInfo)
    },
    editExtraList: async function (type) {
      type = type[0].toUpperCase() + type.toLowerCase().slice(1)
      return shell.openItem(path.join(vrouter.cfgDirPath, vrouter.config.firewallInfo.lists[`extra${type}ListFname`]))
    },
    newProfile: function () {
      // 编辑配置: index >= 0; 新建配置: index = -1; 导入配置: index = -2
      this.editingClone.index = -1
      this.showProfileEditor = true
      console.log('new profile')
    },
    importProfile: function () {
      // 编辑配置: index >= 0; 新建配置: index = -1; 导入配置: index = -2
      this.editingClone.index = -2
      this.showProfileImporter = false
      this.showProfileEditor = true
      console.log('import profile')
    },
    editProfile: function (index) {
      this.editingClone = Object.assign({}, vrouter.config.profiles[index])
      // 编辑配置: index >= 0; 新建配置: index = -1; 导入配置: index = -2
      this.editingClone.index = index
      this.showProfileEditor = true
    },
    applyProfile: async function (index) {
      this.activeLoader = true
      this.activedProfile.active = false
      this.profiles[index].active = true
      // 耗时较长的原因在于重启dnsmasq, 设置ipset需要处理很多条目
      await vrouter.applyActivedProfile()
      await this.getProxiesInfo()
      this.activeLoader = false
      console.log('apply profile', index)
      // todo: save to disk
    },
    deleteProfile: function (index) {
      console.log('about to delete index: ', index)
      this.profiles.splice(index, 1)
      console.log('todo: save to disk')
    },
    editorSave: async function () {
      this.loaderText = 'Applying Profile'
      this.activeLoader = true
      console.log('save profile', this.editingClone)
      await Utils.wait(3000)
      this.showProfileEditor = false
      this.loaderText = 'Loading'
      this.activeLoader = false
    }
  },
  async mounted () {
    // vueInstance = this
    this.bus.$on('editExtraList', this.editExtraList)
    this.bus.$on('newProfile', this.newProfile)
    this.bus.$on('openProfileImporter', () => { this.showProfileImporter = true })
    this.bus.$on('importProfile', this.importProfile)
    this.bus.$on('editProfile', this.editProfile)
    this.bus.$on('applyProfile', this.applyProfile)
    this.bus.$on('deleteProfile', this.deleteProfile)
    this.bus.$on('editorCancel', () => { this.showProfileEditor = false })
    this.bus.$on('editorSave', this.editorSave)

    $('.tabular.menu .item').tab()

    this.getSystemInfo()
    await this.getVrouterInfo()
    await this.getProxiesInfo()

    setInterval(async () => {
      // 每三分钟检测一遍状态, 目前和虚拟机直接只要一个ssh连接, 所以暂时不能并发.
      this.getSystemInfo()
      await this.getVrouterInfo()
      await this.getProxiesInfo()
    }, 180000)

    // setTimeout(() => {
    //   vrouter.config.profiles[0].active = false
    //   vrouter.config.profiles[1].active = true
    // }, 3000)
  }
}
</script>

<style>
.ui.attached.segment {
  border: none !important;
}
.ui.item.menu > a.item.active {
  color: #00B5AD;
}
</style>