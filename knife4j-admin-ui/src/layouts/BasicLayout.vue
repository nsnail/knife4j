<template>
  <div class="BasicLayout">
    <a-layout class="ant-layout-has-sider">
      <a-layout-sider :trigger="null" collapsible :collapsed="collapsed" breakpoint="lg" @collapse="handleMenuCollapse" :width="menuWidth" class="sider">
        <div class="knife4j-logo-data" key="logo" v-if="!collapsed">
          <a to="/" style="float:left;">
            <a-select :value="defaultServiceOption" style="width: 280px" :options="serviceOptions" @change="serviceChange">
            </a-select>
          </a>
        </div>
        <div class="knife4j-logo" key="logo" v-if="collapsed">
          <a to="/" style="float:left;" v-if="collapsed">
            <img :src="logo" alt="logo" />
          </a>
        </div>
        <div class="knife4j-menu">
          <a-menu key="Menu" theme="dark" mode="inline" :inlineCollapsed="collapsed" @openChange="handleOpenChange" @select="selected" :openKeys="openKeys" :selectedKeys="selectedKeys" style="padding: 16px 0; width: 100%">
            <ThreeMenu :menuData="MenuData" :collapsed="collapsed" />
          </a-menu>
        </div>
      </a-layout-sider>
      <!-- <SiderMenu :defaultOption="defaultServiceOption" :serviceOptions="serviceOptions" @menuClick='menuClick' :logo="logo" :menuData="MenuData" :collapsed="collapsed" :location="$route" :onCollapse="handleMenuCollapse" :menuWidth="menuWidth" /> -->
      <a-layout>
        <a-layout-header style="padding: 0;background: #fff;    height: 56px; line-height: 56px;">
          <GlobalHeader @searchKey="searchKey" @searchClear="searchClear" :documentTitle="documentTitle" :collapsed="collapsed" :headerClass="headerClass" :currentUser="currentUser" :onCollapse="handleMenuCollapse" :onMenuClick="item => handleMenuClick(item)" />
        </a-layout-header>
        <context-menu :itemList="menuItemList" :visible.sync="menuVisible" @select="onMenuSelect" />
        <a-tabs hideAdd v-model="activeKey" @contextmenu.native="e => onContextmenu(e)" type="editable-card" @change="tabChange" @edit="tabEditCallback" class="knife4j-tab">
          <a-tab-pane v-for="pane in panels" :key="pane.key" :closable="pane.closable">
            <span slot="tab" :pagekey="pane.key">{{ pane.title }}</span>
            <component :is="pane.content" :data="pane" @childrenMethods="childrenEmitMethod">
            </component>
          </a-tab-pane>
        </a-tabs>
        <a-layout-footer style="padding: 0">
          <GlobalFooter />
        </a-layout-footer>
      </a-layout>
    </a-layout>
  </div>
</template>
<script>
//import logo from "@/assets/logo.png";
import logo from "@/core/logo.js";
import SiderMenu from "@/components/SiderMenu";
import GlobalHeader from "@/components/GlobalHeader";
import GlobalFooter from "@/components/GlobalFooter";
import GlobalHeaderTab from "@/components/GlobalHeaderTab";
import { getMenuData } from "./menu";
import KUtils from "@/core/utils";
import SwaggerBootstrapUi from "@/core/Knife4j.js";
import {
  findComponentsByPath,
  findMenuByKey
} from "@/components/utils/Knife4jUtils";
import { urlToList } from "@/components/utils/pathTools";
import ThreeMenu from "@/components/SiderMenu/ThreeMenu";
//????????????
import ContextMenu from "@/components/common/ContextMenu";
import constant from "@/store/constants";

const constMenuWidth = 310;

export default {
  name: "BasicLayout",
  components: {
    SiderMenu,
    GlobalHeader,
    GlobalFooter,
    GlobalHeaderTab,
    ContextMenu,
    ThreeMenu
  },
  data() {
    /* const panes = [
      {
        title: "??????",
        component: "Main",
        content: "Main",
        key: "kmain",
        closable: false
      }
    ]; */
    return {
      i18n:null,
      logo: logo,
      documentTitle: "",
      menuWidth: constMenuWidth,
      headerClass: "knife4j-header-width",
      swagger: null,
      swaggerCurrentInstance: {},
      defaultServiceOption: "",
      serviceOptions: [],
      MenuData: [],
      collapsed: false,
      linkList: [],
      panels: [],
      panelIndex: 0,
      activeKey: "",
      newTabIndex: 0,
      openKeys: [],
      selectedKeys: [],
      status: false,
      menuVisible: false,
      nextUrl:'',
      nextKey:'',
      menuItemList: [
        { key: "1", icon: "caret-left", text: "????????????" },
        { key: "2", icon: "caret-right", text: "????????????" },
        { key: "3", icon: "close-circle", text: "????????????" }
      ]
    };
  },
  beforeCreate() {},
  created() {
    this.initKnife4jSpringUi();
    //this.initKnife4jFront();
    this.initI18n();
  },
  computed: {
    currentUser() {
      return this.$store.state.header.userCurrent;
    },
    cacheMenuData() {
      return this.$store.state.globals.currentMenuData;
    },currentMenuData(){
      return this.$store.state.globals.currentMenuData;
    }, 
    language(){
       return this.$store.state.globals.language;
    }
  },
  updated() {
    this.openDefaultTabByPath();
    //this.selectDefaultMenu();
  },
  mounted() {
    //this.selectDefaultMenu();
  },
  watch: {
    $route() {
      this.watchPathMenuSelect();
    },
    swaggerCurrentInstance() {
      var title = this.swaggerCurrentInstance.title;
      if (!title) {
        title = "Knife4j ????????????";
      }
      this.documentTitle = title;
      window.document.title = title;
    },
    language:function(val,oldval){
      this.initI18n();
      this.updateMenuI18n();
    }
  },
  methods: {
    getCurrentI18nInstance(){
      this.i18n=this.$i18n.messages[this.language];
      return this.i18n;
    },
    initI18n(){
      //??????i18n?????????????????????
      this.getCurrentI18nInstance();
    },
    updateMenuI18n(){
      //??????i18n?????????,?????????????????????
      //console.log("??????i18n?????????,?????????????????????")
      if(KUtils.arrNotEmpty(this.MenuData)){
        this.MenuData.forEach(m=>{
          if(KUtils.checkUndefined(m.i18n)){
            m.name=this.getCurrentI18nInstance().menu[m.i18n];
            if(KUtils.arrNotEmpty(m.children)){
              m.children.forEach(cm=>{
                if(KUtils.checkUndefined(cm.i18n)){
                  cm.name=this.getCurrentI18nInstance().menu[cm.i18n];
                }
              })
            }
          }
        })
      }
    },
    getPlusStatus(){
      //?????????swagger??????
      var url = this.$route.path;
      var plusFlag = false;
      if (url == "/plus") {
        //????????????
        plusFlag = true;
      }
      return plusFlag;
    },
    getI18nFromUrl(){
      var param=this.$route.params;
      var include=false;
      var i18n="zh-CN";
      if(KUtils.checkUndefined(param)){
        var i18nFromUrl=param["i18n"];
        if(KUtils.checkUndefined(i18nFromUrl)){
          var langs=["zh-CN","en-US"];
          if(langs.includes(i18nFromUrl)){
            include=true;
            i18n=i18nFromUrl;
          }
        }
      }
      return {
        include:include,
        i18n:i18n
      }
    },
    initKnife4jSpringUi() {
      //???????????????????????????knife4j-spring-ui?????????,????????????????????????
      var that = this;
      var i18nParams=this.getI18nFromUrl();
      let params = this.$route.params;
      var tmpI18n=i18nParams.i18n;
      if(i18nParams.include){
        //??????????????????
        this.$store.dispatch("globals/setLang", tmpI18n);
        this.$localStore.setItem(constant.globalI18nCache, tmpI18n);
        this.$i18n.locale = tmpI18n;
        this.initSwagger({ Vue: that, plus: this.getPlusStatus(),code:params.code,i18n:tmpI18n,i18nInstance:this.getCurrentI18nInstance() })
      }else{
        //?????????
        //???????????????i18n????????????add by xiaoymin 2020-5-16 09:51:51
        this.$localStore.getItem(constant.globalI18nCache).then(i18n => {
          if(KUtils.checkUndefined(i18n)){
            this.$store.dispatch("globals/setLang", i18n);
            tmpI18n=i18n;
          }
          this.$i18n.locale = tmpI18n;
          this.initSwagger({ Vue: that, plus: this.getPlusStatus(),code:params.code,i18n:tmpI18n,i18nInstance:this.getCurrentI18nInstance() })
        })
      }
     
    },
    initKnife4jFront() {
      //??????????????????Spring-ui?????????,??????????????????????????????knife4j
      var that = this;
      this.initSwagger({
        Vue: that,
        plus: this.getPlusStatus(),
        //??????config???url??????
        configSupport: false,
        //??????url??????,?????????????????????
        url: "/static/services.json"
      });
    },
    initSwagger(options){
      this.i18n=options.i18nInstance;
      this.swagger = new SwaggerBootstrapUi(options);
      try {
        this.swagger.main();
      } catch (e) {
        console.error(e);
      }
      //?????????????????????
      //?????????????????????
      //this.MenuData = getMenuData();
      //????????????
      this.$store.dispatch("header/getCurrentUser");
    },
    mouseMiddleCloseTab(e) {
      //??????????????????tab??????
      console.log("??????????????????tab??????");
      console.log(e);
    },
    searchClear() {
      //?????????????????????,????????????
      this.MenuData = this.currentMenuData;
    },
    searchKey(key) {
      //??????????????????
      if (KUtils.strNotBlank(key)) {
        var tmpMenu = [];
        var regx = ".*?" + key + ".*";
        //console.log(this.cacheMenuData);
        this.cacheMenuData.forEach(function(menu) {
          if (KUtils.arrNotEmpty(menu.children)) {
            //??????children
            var tmpChildrens = [];
            menu.children.forEach(function(children) {
              var urlflag = KUtils.searchMatch(regx, children.url);
              var sumflag = KUtils.searchMatch(regx, children.name);
              var desflag = KUtils.searchMatch(regx, children.description);
              if (urlflag || sumflag || desflag) {
                tmpChildrens.push(children);
              }
            });
            if (tmpChildrens.length > 0) {
              var tmpObj = {
                groupName: menu.groupName,
                groupId: menu.groupId,
                key: menu.key,
                name: menu.name,
                icon: menu.icon,
                path: menu.path,
                hasNew: menu.hasNew,
                authority: menu.authority,
                children: tmpChildrens
              };
              if(tmpMenu.filter(t=>t.key===tmpObj.key).length==0){
                tmpMenu.push(tmpObj);
              }
            }
          }
        });
        console.log(tmpMenu)
        this.MenuData = tmpMenu;
      }
    },
    serviceChange(value, option) {
      //console("??????????????????");
      var that = this;
      //id
      let swaggerIns = this.swagger.selectInstanceByGroupId(value);
      this.swagger.analysisApi(swaggerIns);
      this.defaultServiceOption = value;
      //console(value);
      //console(option);
      setTimeout(() => {
        that.updateMainTabInstance();
      }, 500);
    },
    onMenuSelect(key, target) {
      let pageKey = this.getPageKey(target);
      switch (key) {
        case "1":
          this.closeLeft(pageKey);
          break;
        case "2":
          this.closeRight(pageKey);
          break;
        case "3":
          this.closeOthers(pageKey);
          break;
        default:
          break;
      }
    },
    onContextmenu(e) {
      const pagekey = this.getPageKey(e.target);
      if (pagekey !== null) {
        e.preventDefault();
        this.menuVisible = true;
      }
    },
    getPageKey(target, depth) {
      depth = depth || 0;
      if (depth > 2) {
        return null;
      }
      let pageKey = target.getAttribute("pagekey");
      pageKey =
        pageKey ||
        (target.previousElementSibling
          ? target.previousElementSibling.getAttribute("pagekey")
          : null);
      return (
        pageKey ||
        (target.firstElementChild
          ? this.getPageKey(target.firstElementChild, ++depth)
          : null)
      );
    },
    closeOthers(pageKey) {
      //??????????????????????????????key??????kmain
      this.linkList = ["kmain", pageKey];
      let tabs = [];
      this.panels.forEach(function(panel) {
        if (panel.key == "kmain" || panel.key == pageKey) {
          tabs.push(panel);
        }
      });
      this.panels = tabs;
      this.activeKey = pageKey;
    },
    closeLeft(pageKey) {
      //????????????,?????????????????????close
      if (this.linkList.length > 2) {
        let index = this.linkList.indexOf(pageKey);
        let sliceArr = this.linkList.slice(index);
        let nLinks = ["kmain"].concat(sliceArr);
        this.linkList = nLinks;
        let kmainComp = this.panels[0];
        let tabs = [];
        tabs.push(kmainComp);
        let splicTabs = this.panels.slice(index);
        this.panels = tabs.concat(splicTabs);
        this.activeKey = pageKey;
      }
    },
    closeRight(pageKey) {
      this.activeKey = pageKey;
      let index = this.linkList.indexOf(pageKey);
      let tmpLinks = [];
      let tmpTabs = [];
      const tpLinks = this.linkList;
      const tpPanels = this.panels;
      for (var i = 0; i <= index; i++) {
        tmpLinks.push(tpLinks[i]);
        tmpTabs.push(tpPanels[i]);
      }
      this.linkList = tmpLinks;
      this.panels = tmpTabs;
    },
    childrenEmitMethod(type, data) {
      this[type](data);
    },
    addGlobalParameters(data) {
      this.swaggerCurrentInstance.globalParameters.push(data);
    },
    getDefaultBrowserPath(){
      var url = this.$route.path;
      //console.log("????????????url:"+url)
      //i18n?????????,?????????
      if(url.startWith("/plus")){
        url="/plus";
      }
      if(url.startWith("/home")){
        url="/home";
      }
      if (url == "/plus") {
        //??????????????????????????????,?????????????????????????????????
        url = "/home";
      }
      return url;
    },
    openDefaultTabByPath() {
      //?????????????????????Tab?????????
      var that = this;
      //console.log("openDefaultTabByPath")
      const panes = this.panels;
      /* var url = this.$route.path;
      console.log("????????????url:"+url)
      if (url == "/plus") {
        //??????????????????????????????,?????????????????????????????????
        url = "/home";
      } */
      var url=this.getDefaultBrowserPath();
      if(this.nextUrl===url){
         //this.activeKey = this.nextKey;
         //console.log("???????????????,????????????2???")
      }else{
        //console.log("url:"+url)
        //var menu = findComponentsByPath(url, this.MenuData);
        var menu = findComponentsByPath(url, this.swagger.globalMenuDatas);
        if (menu != null) {
          //?????????????????????????????????????????????
          const indexSize = this.panels.filter(tab => tab.key == "kmain");
          if (indexSize == 0) {
            panes.push({
              /* title: "??????", */
              title: this.getCurrentI18nInstance().menu.home,
              component: "Main",
              content: "Main",
              key: "kmain",
              instance: this.swaggerCurrentInstance,
              closable: false
            });
            this.linkList.push("kmain");
          }
          const tabKeys = panes.map(tab => tab.key);
          //console.log(tabKeys)

          //??????tab???????????????
          if (tabKeys.indexOf(menu.key) == -1) {
            //console.log("??????-?????????")
            //console(menu);
            //console(this.swaggerCurrentInstance);
            //?????????,???????????????????????????tab??????
            panes.push({
              title: menu.tabName ? menu.tabName : menu.name,
              content: menu.component,
              key: menu.key,
              instance: this.swaggerCurrentInstance,
              closable: menu.key != "kmain"
            });
            this.linkList.push(menu.key);
            this.panels = panes;
          }
          this.activeKey = menu.key;
          this.nextUrl=url;
          this.nextKey=menu.key;
          //console.log("??????menu1")
        } else {
          //??????
          this.activeKey = "kmain";
          //this.nextUrl=url;
          this.nextKey="kmain";
          this.updateMainTabInstance();
          //console.log("??????menu2")
        }

      }
      //this.watchPathMenuSelect();
    },
    updateMainTabInstance() {
      var that = this;
      //??????kmain??????instance????????????
      that.panels.forEach(function(panel) {
        if (panel.key == "kmain") {
          panel.instance = that.swaggerCurrentInstance;
        }
      });
    },
    watchPathMenuSelect() {
      var url = this.$route.path;
      const tmpcol = this.collapsed;
      //console.log("watch-------------------------");
      const pathArr = urlToList(url);
      //console(pathArr);
      var m = findComponentsByPath(url, this.MenuData);
      //???????????????????????????,???????????????openKeys
      if (!tmpcol) {
        if (pathArr.length == 5) {
          //console.log("??????????????????"+pathArr[3])
          //???????????????
          var parentM = findComponentsByPath(pathArr[3], this.MenuData);
          if (parentM != null) {
            this.openKeys = [parentM.key];
          }
        } else if (pathArr.length == 6) {
          //???????????????
          var parentM = findComponentsByPath(pathArr[4], this.MenuData);
          if (parentM != null) {
            this.openKeys = [parentM.key];
          }
        } else {
          if (m != null) {
            this.openKeys = [m.key];
          }
        }
      }

      //this.selectedKeys = [this.location.path];
      if (m != null) {
        this.selectedKeys = [m.key];
      }
    },
    selectDefaultMenu() {
      var url = this.$route.path;
      const pathArr = urlToList(url);
      //console.log('selectDefaultMenu')
      var m = findComponentsByPath(url, this.MenuData);
      if (pathArr.length == 5) {
        //???????????????
        var parentM = findComponentsByPath(pathArr[3], this.MenuData);
        if (parentM != null) {
          this.openKeys = [parentM.key];
        }
      } else {
        this.openKeys = [m.key];
      }
      //this.selectedKeys = [this.location.path];
      if (m != undefined && m != null) {
        this.selectedKeys = [m.key];
      }
    },
    menuClick(key) {
      //console("??????click");
      //console(key);
      const panes = this.panels;
      //console(panes);
      const tabKeys = this.panels.map(tab => tab.key);
      // var menu = findComponentsByPath(url, this.MenuData);
      var menu = findMenuByKey(key, this.MenuData);
      //console(menu);
      if (menu != null) {
        //??????tab???????????????
        if (tabKeys.indexOf(menu.key) == -1) {
          //?????????,???????????????????????????tab??????
          panes.push({
            title: menu.name,
            content: menu.component,
            key: menu.key,
            closable: true
          });
          this.linkList.push(menu.key);
          this.panels = panes;
        }
        this.activeKey = menu.key;
      } else {
        //??????
        this.activeKey = "kmain";
        this.updateMainTabInstance();
      }
    },
    tabEditCallback(targetKey, action) {
      this[action](targetKey);
    },
    tabChange(targetKey) {
      //console("tabchange------------");
      //console(targetKey);
      //var menu = findMenuByKey(targetKey, this.MenuData);
      var menu = findMenuByKey(targetKey, this.swagger.globalMenuDatas);
      //console(menu);
      if (menu != null) {
        var path = menu.path;
        this.$router.push({ path: path });
      } else {
        this.$router.push({ path: "/" });
      }
    },
    remove(targetKey) {
      let activeKey = this.activeKey;
      const flag = targetKey == activeKey;
      let lastIndex;
      this.panels.forEach((pane, i) => {
        if (pane.key === targetKey) {
          lastIndex = i - 1;
        }
      });
      const panes = this.panels.filter(pane => pane.key !== targetKey);
      if (panes.length && activeKey === targetKey) {
        if (lastIndex >= 0) {
          activeKey = panes[lastIndex].key;
        } else {
          activeKey = panes[0].key;
        }
      }
      this.panels = panes;
      this.activeKey = activeKey;
      //????????????????????????
      if (flag) {
        this.tabChange(activeKey);
      }
    },
    handleMenuCollapse(collapsed) {
      const tmpColl = this.collapsed;
      this.collapsed = !tmpColl;
      //console("??????selectDefaultMenu");
      this.selectDefaultMenu();
      setTimeout(() => {
        if (tmpColl) {
          this.headerClass = "knife4j-header-width";
          this.menuWidth = constMenuWidth;
        } else {
          this.headerClass = "knife4j-header-width-collapsed";
          this.menuWidth = 80;
          //this.openKeys = [""];
        }
      }, 10);
    },
    handleOpenChange(openKeys) {
      let keys;
      if (openKeys.length > 1) {
        if (openKeys.length > 2) {
          keys = [openKeys[openKeys.length - 1]];
        } else if (openKeys[1].indexOf(openKeys[0]) > -1) {
          keys = [openKeys[0], openKeys[1]];
        } else {
          keys = [openKeys[openKeys.length - 1]];
        }
        this.openKeys = keys;
      } else {
        this.openKeys = openKeys;
      }
    },
    // eslint-disable-next-line
    selected({ item, key, selectedKeys }) {
      this.selectedKeys = selectedKeys;
    },
    // eslint-disable-next-line
    collapsedChange(val, oldVal) {
      // eslint-disable-line
      /* if (val) {
        this.openKeys = [];
      } else {
        const pathArr = urlToList(this.location.path);
        if (pathArr[2]) {
          this.openKeys = [pathArr[0], pathArr[1]];
        } else {
          var m = findComponentsByPath(pathArr[0], this.menuData);
          //this.openKeys = [pathArr[0]];
          this.openKeys = [m.key];
        }
      } */
    }
  }
};
</script>

<style lang="less" scoped></style>
