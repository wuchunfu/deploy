<template>
    <div>
        <el-form ref="searchForm" :model="searchForm" :inline="true">
            <el-form-item label="筛选条件" prop="condition">
                <el-select style="width: 120px" v-model="searchForm.sCondition" placeholder="请选择">
                    <el-option key="task_name" label="任务名称" value="env_name"/>
                </el-select>
            </el-form-item>
            <el-form-item prop="searchValue">
                <el-input v-model="searchForm.sValue" placeholder="搜索关键词"></el-input>
            </el-form-item>
            <el-form-item>
                <el-button @click="getTableData(1)" type="primary">查询</el-button>
            </el-form-item>
        </el-form>
        <div class="button-box">
            <el-button @click="editTask"><i class="el-icon-plus"></i> 新增任务</el-button>
        </div>
        <el-table
            :data="tableData"
            border
            style="width: 100%">
            <el-table-column
                prop="task_id"
                fixed
                label="ID"
                width="60">
                <template slot-scope="scope">
                    <span style="color: red; margin-right: 10px"
                          v-if="scope.row.uuid === scope.row.EnvCfg.uuid">☆</span>
                    <span v-else></span><!-- 解决当前面v-if为false导致当前template不渲染的bug -->
                    <span>{{ scope.row.task_id }}</span>
                </template>
            </el-table-column>
            <el-table-column
                fixed
                width="120"
                label="任务标题">
                <template slot-scope="scope">
                    <el-tooltip class="item" v-if="scope.row.description!==''" effect="dark"
                                :content="scope.row.description" placement="top">
                        <span>{{ scope.row.task_name }}</span>
                    </el-tooltip>
                    <span v-else>{{ scope.row.task_name }}</span>
                </template>
            </el-table-column>
            <el-table-column
                fixed
                width="120"
                prop="EnvCfg.env_name"
                label="环境">
            </el-table-column>
            <el-table-column
                :formatter="deployTypeFormatter"
                fixed
                width="70"
                label="发布类型">
            </el-table-column>
            <el-table-column
                fixed
                prop="branch"
                width="150"
                label="分支">
            </el-table-column>
            <el-table-column
                prop="version"
                width="100"
                label="版本">
            </el-table-column>
            <el-table-column
                prop="uuid"
                min-width="280"
                label="UUID">
            </el-table-column>
            <el-table-column
                prop="create_at"
                width="210"
                label="创建时间">
            </el-table-column>
            <el-table-column
                prop="update_at"
                width="210"
                label="更新时间">
            </el-table-column>
            <el-table-column
                width="80"
                fixed="right"
                label="状态">
                <template slot-scope="scope">
                    <el-popover v-if="scope.row.status === 8 || scope.row.status === 9"
                                placement="left"
                                trigger="hover"
                                width="800">
                        <TaskInfo :data="scope.row.output" :status="status"></TaskInfo>
                        <el-tag slot="reference" v-if="scope.row.status === 9"
                                type="danger" size="mini">{{ status[scope.row.status] }}
                        </el-tag>
                        <el-tag slot="reference" v-else type="success"
                                size="mini">{{ status[scope.row.status] }}
                        </el-tag>
                    </el-popover>
                    <el-popover v-else-if="scope.row.status === 2" placement="left"
                                trigger="hover"
                                width="800">
                        <ProcessInfo
                            :process-data="taskProcess"
                            :com-process="getCommonProcess(scope.row)"
                            :task-id="scope.row.task_id" :server-process="server"></ProcessInfo>
                        <el-tag slot="reference" size="mini">{{ status[scope.row.status] }}</el-tag>
                    </el-popover>
                    <el-tag slot="reference" v-else size="mini">{{ status[scope.row.status] }}</el-tag>
                </template>
            </el-table-column>
            <el-table-column align="center" width="120" fixed="right" label="操作">
                <template slot-scope="scope">
                    <el-tooltip v-if="scope.row.status === 1" content="发布" placement="top" effect="dark">
                        <el-button @click="deployTask(scope.row)" type="primary" icon="el-icon-upload"
                                   circle></el-button>
                    </el-tooltip>
                    <span v-else></span><!-- 解决当前面v-if为false导致当前template不渲染的bug -->
                    <el-button v-if="scope.row.status === 1 || scope.row.status === 9" type="danger"
                               icon="el-icon-delete" circle
                               @click="delTask(scope.row)"
                               slot="reference"></el-button>
                    <el-tooltip v-if="scope.row.status === 8 && scope.row.uuid !== ''
                     && scope.row.uuid !== scope.row.EnvCfg.uuid" content="回滚" placement="top"
                                effect="dark">
                        <el-button type="primary"
                                   icon="el-icon-refresh-left" circle
                                   @click="rollBackTask(scope.row)"></el-button>
                    </el-tooltip>
                </template>
            </el-table-column>
        </el-table>
        <div class="page-content">
            <el-pagination
                @size-change="handleSizeChange"
                @current-change="handleCurrentChange"
                :current-page="currentPage"
                :page-sizes="[10, 20, 50, 100]"
                :page-size="pageSize"
                layout="total, sizes, prev, pager, next, jumper"
                :total="total">
            </el-pagination>
        </div>
        <TaskEdit :d-type="dType" ref="taskEditorFormDrawer"></TaskEdit>
    </div>
</template>

<script>
import tableInfo from "@/plugins/mixins/tableInfo";
import TaskEdit from "./cpns/TaskEdit";
import TaskInfo from "./cpns/TaskInfo";
import ProcessInfo from "./cpns/ProcessInfo"
import {deleteTask, deployTask, getTaskList, rollBackTask} from "@/api/task";
import {InitWebSocket} from "@/utils/websocket";

export default {
    name: "TaskList",
    mixins: [tableInfo],
    components: {TaskEdit, TaskInfo, ProcessInfo},
    created() {
        this.getTableData()
    },
    data() {
        return {
            dType: [
                {
                    label: "增量",
                    value: 1
                },
                {
                    label: "全量",
                    value: 2
                },
            ],
            status: {
                '1': '未开始',
                '2': '发布中',
                '8': '成功',
                '9': '失败'
            },
            direct: [   //直发公共
                {
                    k: 'pack',
                    v: '打包'
                },
            ],
            jumper: [   //跳发公共
                {
                    k: 'pack',
                    v: '打包'
                },
                {
                    k: 'upload_jumper',
                    v: '上传跳板'
                },
            ],
            server: [   //直发/跳发 非公共
                {
                    k: 'upload_dst',
                    v: '上传包'
                },
                {
                    k: 'deploy',
                    v: '发布'
                },
                {
                    k: 'change_dir',
                    v: '切目录'
                },
            ],
            taskProcess: {},
            taskConnected: []
        }
    },
    methods: {
        getList: getTaskList,
        async deployTask(row) {
            this.$confirm('确定发布?', '提示', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
            }).then(async () => {
                await deployTask({task_id: row.task_id}).then((res) => {
                    this.$message({
                        type: 'success',
                        message: res.msg
                    })
                    this.getTableData()
                }).catch(() => {
                })
            }).catch(() => {
                this.$message({
                    type: 'info',
                    message: '已取消'
                })
            })
        },
        editTask() {
            this.$refs.taskEditorFormDrawer.setEditVal()
        },
        rollBackTask(row) {
            this.$confirm('确认回滚到该工作目录?', '提示', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
            }).then(async () => {
                await rollBackTask({task_id: row.task_id}).then((res) => {
                    let isFail = Object.keys(res.data.error).length > 0
                    this.$message({
                        type: isFail ? 'error' : 'success',
                        message: isFail ? '回滚失败(可能部分失败), 原因:' + JSON.stringify(res.data) : '回滚成功'
                    })
                    this.getTableData()
                }).catch(() => {
                })
            }).catch(() => {
                this.$message({
                    type: 'info',
                    message: '已取消'
                })
            })
        },
        delTask(row) {
            this.$confirm('确认删除该任务?', '提示', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
            }).then(async () => {
                await deleteTask({task_id: row.task_id}).then((res) => {
                    this.$message({
                        type: 'success',
                        message: res.msg
                    })
                    this.getTableData()
                }).catch(() => {
                })
            }).catch(() => {
                this.$message({
                    type: 'info',
                    message: '已取消'
                })
            })
        },
        deployTypeFormatter(row) {
            return this.dType.reduce((o, n) => {
                return n.value === row.deploy_type ? n.label : o;
            }, '')
        },
        getCommonProcess(row) {
            if (row.EnvCfg.Jumper.server_id !== 0) {
                return this.jumper
            } else {
                return this.direct
            }
        },
        connectWs(task_id) {
            this.taskConnected.push(task_id)
            InitWebSocket(task_id, async () => {
                await this.getTableData()
                this.$delete(this.taskProcess, task_id)
                if (this.taskConnected.indexOf(task_id) !== -1) {
                    this.taskConnected.splice(this.taskConnected.indexOf(task_id), 1)
                }
            }, (d) => {
                this.UpdateProcessBar(d)
            })
        },
        UpdateProcessBar(resp) {
            let _data = resp['data'], _comProcess = 1, _pStatus = 'success'
            for (let v of this.jumper) {
                if (_data[v.k] !== undefined) {
                    if (_data[v.k] === 'success') {
                        _comProcess++
                    } else {
                        _pStatus = 'error'
                    }
                }
            }
            this.taskProcess[resp.taskId]['now_step'] = _comProcess;
            this.taskProcess[resp.taskId]['now_status'] = _pStatus;
            for (let ip in _data['servers']) {
                let _process = 0, _sStatus = 'success'
                for (let _pk of this.server) {
                    if (_data['servers'][ip][_pk.k] !== undefined) {
                        if (_data['servers'][ip][_pk.k] === 'success') {
                            _sStatus = 'process'
                            _process++
                        } else {
                            _sStatus = 'error'
                        }
                    }
                }
                this.taskProcess[resp.taskId]['servers'][ip] = {
                    now_step: _process,
                    now_status: _sStatus
                }
            }
        },
        afterUpdateList() {
            this.tableData.map((t) => {
                if (t.status === 2 && this.taskConnected.indexOf(t.task_id) === -1) {
                    this.$set(this.taskProcess, t.task_id, {})
                    this.$set(this.taskProcess[t.task_id], 'now_step', 1)
                    this.$set(this.taskProcess[t.task_id], 'now_status', 'process')
                    this.$set(this.taskProcess[t.task_id], 'servers', {})
                    for (let srv of t['EnvCfg']['Servers']) {
                        this.$set(this.taskProcess[t.task_id]['servers'], srv['ssh_addr'], {
                            now_step: 0,
                            now_status: 'wait'
                        })
                    }
                    this.connectWs(t.task_id)
                }
            })
        }
    },
}
</script>

<style scoped>

</style>
