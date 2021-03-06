<template>
    <card-container>
        <app-tables :table="table" :utils="table.utils" @pageChange="handleCurrentChange" @selectionChange="handleSelectionChange">
            <template>
                <el-button v-if="$store.state.permission.includes('add')" slot="add" type="primary" icon="el-icon-circle-plus" plain @click="showDialog({type:'add'})">
                    上传文件
                </el-button>
                <el-select v-if="$store.state.permission.includes('select')" slot="select" v-model="table.queryData.isDelete" placeholder="文件筛选" @change="init()">
                    <el-option label="全部" :value="null" />
                    <el-option label="正常" :value="0" />
                    <el-option label="已标记为删除" :value="1" />
                </el-select>
                <el-button
                    v-if="$store.state.permission.includes('delete')"
                    slot="delete"
                    plain
                    type="danger"
                    icon="el-icon-delete"
                    :disabled="!deleteData.ids.length"
                    @click="handleDel(deleteData)"
                >
                    批量删除
                </el-button>
                <!-- <el-button type="text" @click="handleDel({type:1})">
                    删除
                </el-button> -->
                <!-- <el-button slot="export" plain @click="excelAllDownload()">
                    导出
                </el-button> -->
            </template>
            <template slot="column" slot-scope="{data}">
                <template v-if="data.col.key === 'operation'">
                    <el-button-group>
                        <!-- <el-button title="编辑" type="warning" icon="el-icon-edit" @click="showDialog({type:'update',data:data.row})" /> -->
                        <el-button v-if="$store.state.permission.includes('delete')" title="删除" type="danger" icon="el-icon-delete" @click="handleDel(data.row)" />
                    </el-button-group>
                </template>
                <template v-else-if="data.col.key === 'size'">
                    {{ data.row[data.col.key] | formatFileSize }}
                </template>
                <template v-else-if="data.col.key === 'view'">
                    <template v-if="['image/jpeg','image/png','image/gif'].includes(data.row.type)">
                        <el-image
                            fit="cover"
                            :src="data.row.path"
                            :preview-src-list="[data.row.path]"
                        />
                    </template>
                    <template v-else-if="['text/plain'].includes(data.row.type)">
                        <read-file-dialog :path="data.row.path" />
                    </template>
                    <template v-else>
                        暂不支持预览
                    </template>
                </template>
                <template v-else-if="data.col.key === 'status'">
                    <el-tag v-if="data.row[data.col.key] " type="warning">
                        禁用
                    </el-tag>
                    <el-tag v-else-if="data.row.isDelete" type="danger">
                        已删除
                    </el-tag>
                    <el-tag v-else type="success">
                        正常
                    </el-tag>
                </template>
                <template v-else>
                    {{ data.row[data.col.key] || '-' }}
                </template>
            </template>
        </app-tables>
        <file-form ref="FileForm" @refresh="init()" />
    </card-container>
</template>
<script>
import { getFiles, deleteFiles } from '@/http';
import FileForm from './FileForm';
export default {
    head: {
        title: '文件管理'
    },
    components: { FileForm },
    data () {
        return {
            table: {
                queryData: {
                    keyword: null,
                    isDelete: null, //筛选被删除的文件
                    page: 1, //获取第几页的数据，默认为1
                    limit: 10//每页数据条数，默认为10
                },
                data: [], //表格数据
                total: 0, //总页数
                tableType: 1, //表格类型
                utils: { //表格工具栏
                    left: [{ slot: 'add' }, { slot: 'delete' }, { slot: 'select' }],
                    right: [{ slot: 'search' }],
                    cols: [12, 12]
                },
                cols: [ //表格列配置
                    {
                        key: 'userName',
                        label: '上传用户'
                    },
                    {
                        key: 'fileName',
                        label: '文件名'
                    },
                    {
                        key: 'suffix',
                        label: '后缀名'
                    },
                    {
                        key: 'size',
                        label: '大小'
                    },
                    {
                        key: 'type',
                        label: '类型'
                    },
                    {
                        key: 'view',
                        label: '预览',
                        overflow: true
                    },
                    {
                        key: 'status',
                        label: '状态'
                    },
                    {
                        key: 'path',
                        label: '路径'
                    },
                    {
                        key: 'aliasName',
                        label: '别名'
                    },
                    {
                        key: 'remark',
                        label: '备注'
                    },
                    {
                        key: 'operation',
                        width: '180px',
                        label: '操作'
                    }
                ]
            },
            deleteData: {
                ids: []
            }
        };
    },
    created() {
        this.init();
        console.log(this.$store.state.permission);
    },

    methods: {

        async init () {
            try {
                const { data: { total, list } } = await this.$axios[getFiles.method](getFiles.url, { params: this.table.queryData });
                this.table.data = list;
                this.table.total = total;
            } catch (error) {
                console.error(error);
            }
        },

        /**
         * 分页器回调
         */
        handleCurrentChange(page) {
            this.table.queryData.page = page;
            this.init();
        },

        /**
	     * 弹出层控制
	     */
        showDialog ({ type, data }) {
            if (type === 'add') {
                this.$refs.FileForm.init({ type, title: '上传文件' });
            } else if (type === 'update') {
                this.$refs.FileForm.init({ type, data, title: '编辑文件' });
            }
        },

        /**
         * 删除
         */
        handleDel ({ fileId, ids }) {
            this.$confirm(`此操作将会删除文件,是否继续?`, '提示', {
                cancelButtonText: '取消',
                confirmButtonText: '确定',
                type: 'warning',
                center: true,
                customClass: 'bg-warning'
            }).then(async () => {
                const { code } = await this.$axios[deleteFiles.method](deleteFiles.url, {
                    data: { ids: ids || [fileId], isDelete: true }
                });
                if (code == 200) {
                    this.$message.success(this.$t('msg.deleted_success'));
                    this.init();
                }
            }).catch(() => {});
        },

        /**
         * 批量删除
         */
        handleSelectionChange(list) {
            this.deleteData.ids = list.map(item => item.fileId);
        }
    }
};
</script>
