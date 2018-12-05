<style lang="less">
    @import './VTable.less';
</style>

<template>
    <div class="VTable">
        <Row>
            <Col>
                <Table :columns="init.cols" stripe ref="selection" @on-sort-change="toChangeSort"
                       no-data-text="<i class='iconfont finleanCloud-wushuju'></i><p class='noData'>无数据</p>"
                       @on-selection-change="onSelect" :data="tableData" :loading='showLoading'></Table>
            </Col>
        </Row>
        <br>
        <Row type="flex" justify="space-between" v-if="pages">
            <Col style="color: #5c92e8;height: 32px;font-size: 14px; line-height: 32px;">
                <span>共 {{ totleItems }} 条</span>
                &nbsp;&nbsp;&nbsp;&nbsp;
                <span v-if="showExportPower" @click="exportTable" style="cursor:pointer">全部导出</span>
            </Col>
            <Col>
                <Page :total="totalPage" placement="top" :current="currentPage" :page-size='rows' show-sizer
                      show-elevator :page-size-opts="pageSizeOpts" @on-change='changePage'
                      @on-page-size-change="changePageNum"></Page>
            </Col>
        </Row>
    </div>
</template>

<script>
    /**
     * 2018.2.4
     * @author TANGiMING
     * <VTable></VTable>
     * @description 一个表格+分页组件，在组件上使用:init="Object"来进行表格初始化
     * @event @changed  选填，表格渲染后触发的事件
     * @prop {Boolean} pages 选填，是否需要分页,默认为true
     * 以下参数为Object中的配置项
     * @prop {String} url 必填，API方法名,表格会自动发送请求并自动渲染,例如:'fetchINSFOrderList'
     * @prop {Array} cols 必填，列，用法和iview的表格一致，可以使用render函数
     * @prop {Object} defaultValue  选填，为表格开始渲染时需要传递的默认值（除了Page和rows之外的其他值）
     * @prop {Object} filt 选填，需要主动更新表格时使用，会带入filt的属性进行请求，使用watch监听
     * @prop {Object} exportExcelUrl ,选填，两个属性：
     *      URL：导出的API，
     *      fileName：导出的文件名
     * @prop {Object} exportPower,选填，控制导出权限
     *      show：是否显示导出按钮，
     *      btnId：权限的id
     * @method getData 更新表格，参数传递defaultValue表明需要携带的参数
     **/
    /*
    *
    * 2018-6-25
    * 新增了表格自定义筛选的功能：
    * 通过给需要筛选的列，设置sortKey来自定义排序的key，点击升序和降序的图标时会自动传递0和1来进行排序
    *
    * 2018-6-27
    * 新增了导出权限的功能
    * init下的exportExcelUrl为导出的Url
    * exportExcelUrl:{
    *   URL:''  导出的地址
    *   fileName:''  导出的文件名
    * }
    * 在init中通过添加exportPower对象来进行判断，该对象有两个属性
    * @prop show 是否显示导出功能
    * @prop btnId 权限id
    * 如果show为false，则不显示
    * 如果show为true，则判断btnId是否传递，如果没有传递，则显示，如果传递则判断是否有权限显示，如果有，则显示
    *
    * */
    import api from '../../api/api';
    import codeManage from '@/api/statusCodeManage';
    import downloadUrl from '@/api/excelDownLoadConfig';

    export default {
        name: 'VTable',
        //        props: ['init', 'pages', 'filt', 'defaultValue', 'rows'],
        props: {
            init: {
                type: Object,
                default: () => {
                    return {};
                }
            },
            pages: {
                type: Boolean,
                default: true
            },
            filt: {
                type: Object,
                default: () => {
                    return {};
                }
            },
            defaultValue: {
                type: Object,
                default: () => {
                    return {};
                }
            },
            rows: {
                type: Number,
                default: 20
            }
        },
        data() {
            return {
                showExportPower: false,
                tableData: [],
                showLoading: true,
                totleItems: 0,
                totalPage: null, // 总页数
                pageSizeOpts: [5, 10, 20], //每页条数切换的配置
                auditArr: [],
                currentPage: 1
                // sortType: 2
            };
        },
        watch: {
            filt(newVal) {
                this.getData(newVal);
            }
        },
        methods: {
            exportTable() {
                if (!this.tableData.length) {
                    this.$Notice.error({title: '当前无数据导出！'});
                    return;
                }
                if (!this.init.hasOwnProperty('exportExcelUrl')) {
                    throw 'before export excel , you must set property "exportExcelUrl[Object]" as the base interface';
                    return;
                }
                if (
                    typeof this.init.exportExcelUrl != 'object' ||
                    !(this.init.exportExcelUrl instanceof Object)
                ) {
                    throw 'exportExcelUrl must be an Object , not Array Number String!';
                    return;
                }
                if (!this.init.exportExcelUrl.url) {
                    throw 'you have to set url as the interface for excel download';
                    return;
                }
                let params = {...this.init.exportExcelUrl};
                let _url = params.url.split('.');
                if (!params.hasOwnProperty('fileName')) {
                    params.fileName = '';
                }
                api[_url[0]][_url[1]]({...this.defaultValue}).then(
                    res => {
                        downloadUrl.configDate(res, params.fileName);
                    },
                    err => {
                        codeManage.showTipOfStatuCode(err, this);
                    }
                );
            },
            toChangeSort(val) {
                //排序功能，点击排序按钮时触发
                //                let toSort = {
                //                    ...this.defaultValue,
                //                    rows: this.rows
                //                };
                this.defaultValue.rows = this.rows;
                if (val.order === 'desc') {
                    this.defaultValue[val.column.sortKey] = 1;
                } else if (val.order === 'asc') {
                    this.defaultValue[val.column.sortKey] = 0;
                }
                this.defaultValue.page = this.currentPage;
                this.getData(this.defaultValue);
            },
            onSelect(selection, row) {
                //批量操作
                let arr = [];
                selection.forEach(val => {
                    arr.push(val);
                });
                this.$emit('selectArr', arr);
            },
            getData(params) {
                //if api is undefined
                if (!this.init.url) {
                    throw 'Please set property "url" as interface url ';
                }
                //open loading
                this.showLoading = true;
                //if params is not a Object
                typeof params != 'object' && (params = {});
                //if params is an empty Object or no pageNum
                if (
                    JSON.stringify(params) === '{}' ||
                    !params.hasOwnProperty('page')
                ) {
                    // 判读空对象或对象里面没有传page和rows时
                    params.page = 1;
                    params.rows = this.rows;
                }
                // if (!params.hasOwnProperty('page')) {
                //     params.page = 1;
                // }
                // if (!params.hasOwnProperty('rows')) {
                //     params.rows = this.rows;
                // }
                // params.sortType = this.sortType;
                //main function
                let _url = this.init.url.split('.');
                api[_url[0]][_url[1]]({...params}).then(
                    res => {
                        this.totalPage = res.body.total;
                        this.pageSize = res.body.rows;
                        res.body.page === 0
                            ? (this.currentPage = 1)
                            : (this.currentPage = res.body.page);
                        this.showLoading = false;
                        this.totleItems = res.body.total || 0;
                        this.tableData = res.body.items || [];
                        this.$emit('changed', res.body.total, res.body);
                    },
                    err => {
                        codeManage.showTipOfStatuCode(err, this);
                        this.showLoading = false;
                    }
                );
            },
            changePage(currpage) {
                // 点击翻页和直接跳转页
                let changePage = {
                    ...this.defaultValue,
                    rows: this.rows
                };
                changePage.page = currpage;
                //				changePage.sortType=this.sortType;
                this.getData(changePage);
                // this.$emit('changePage', currpage);
            },
            changePageNum(num) {
                // 改变每页显示多少个
                this.rows = num;
                let showRows = {
                    ...this.defaultValue,
                    page: 1,
                    rows: num
                };
                //				showRows.sortType=this.sortType;
                this.getData(showRows);
            }
        },
        created() {
            if (this.init.exportPower !== undefined) {
                if (this.init.exportPower.show) {
                    //两种情况，如果没有传ID 则显示，如果ID判断为真 也显示，只有ID判断为假的时候才不显示
                    if (!this.init.exportPower.btnId) {
                        this.showExportPower = true;
                    } else if (this.filterBtnById(this.init.exportPower.btnId)) {
                        this.showExportPower = true;
                    } else {
                        this.showExportPower = false;
                    }
                } else {
                    this.showExportPower = false;
                }
            } else {
                this.showExportPower = false;
            }
            let defaultVal = this.defaultValue;
            this.getData(defaultVal);
        }
    };
</script>
