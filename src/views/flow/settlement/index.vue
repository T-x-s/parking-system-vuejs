<template>
    <div>
        <el-card class="box-card" header="条件筛选">
            <el-form :inline="true" :model="form" ref="searchRef" label-width="100px" class="demo-form-inline">
                <el-form-item label="车牌号：" prop="plateNumber">
                    <el-input v-model="form.plateNumber" placeholder="请输入车牌号" style="width: 240px;"></el-input>
                </el-form-item>
                <el-form-item label="车位号：" prop="carNumber">
                    <el-input v-model="form.carNumber" placeholder="请输入车位号" style="width: 240px;"></el-input>
                </el-form-item>
                <el-form-item label="车主电话：" prop="phone">
                    <el-input v-model="form.phone" placeholder="请输入车主电话" style="width: 240px;"></el-input>
                </el-form-item>
                <div>
                    <el-form-item label="是否支付：" prop="status">
                        <el-select v-model="form.status" placeholder="请选择支付状态" style="width: 240px;">
                            <el-option label="未支付" :value="2"></el-option>
                            <el-option label="已支付" :value="1"></el-option>
                        </el-select>
                    </el-form-item>
                    <el-form-item>
                        <el-button type="primary" @click="form.pageNum = 1, getList()">搜索</el-button>
                        <el-button @click="cancel('searchRef')">重置</el-button>
                    </el-form-item>
                </div>
            </el-form>
        </el-card>
        <el-card class="box-card mt-24">
            <div slot="header" class="card-center">
                <span>车辆结算列表</span>
                <el-button type="primary" @click="exportExcel" :loading="exportLoading">一键导出</el-button>
            </div>
            <el-table :data="tableData" border style="width: 100%" ref="tableRef" @select="handleSelection"
                @select-all="handleSelectionAll">
                <el-table-column type="selection" width="55"></el-table-column>
                <el-table-column type="index" label="序号" width="50">
                </el-table-column>
                <el-table-column prop="plateNumber" label="车牌号">
                </el-table-column>
                <el-table-column prop="carNumber" label="车位号">
                </el-table-column>
                <el-table-column prop="ownerName" label="车主姓名">
                </el-table-column>
                <el-table-column prop="phone" label="车主电话">
                </el-table-column>
                <el-table-column prop="chargeHour" label="每小时收费(元)">
                </el-table-column>
                <el-table-column prop="exittime" label="入场时间">
                </el-table-column>
                <el-table-column prop="leavingTime" label="离场时间">
                    <template v-slot="{ row, column }">{{ row[column.property] ? row[column.property] : "--" }}</template>
                </el-table-column>
                <el-table-column prop="duration" label="停放时长(小时)">
                    <template v-slot="{ row, column }">{{ row[column.property] ? `${row[column.property]}` : "--" }}</template>
                </el-table-column>
                <el-table-column prop="amount" label="应收费金额(元)">
                    <template v-slot="{ row, column }">{{ row[column.property] ? `${row[column.property]}` : "--" }}</template>
                </el-table-column>
                <el-table-column prop="status" label="是否支付">
                    <template v-slot="{ row, column }">{{ row[column.property] == 1 ? '已支付' : "未支付" }}</template>
                </el-table-column>
                <el-table-column fixed="right" label="操作" width="180" align="center">
                    <template slot-scope="scope">
                        <el-button type="text" @click="handleDeparture(scope.row)"
                            :disabled="scope.row.status == 1">结算</el-button>
                    </template>
                </el-table-column>
            </el-table>
            <!-- 分页 -->
            <pagination v-show="total > 0" :total="total" :page.sync="form.pageNum"
                :limit.sync="form.pageSize" @pagination="getList" />
        </el-card>
        <!-- 结算弹窗 -->
        <el-dialog title="结算离场" :visible.sync="dialogVisible" width="30%">
            <el-descriptions>
                <el-descriptions-item label="当前时间">{{ leavingTime }}</el-descriptions-item>
            </el-descriptions>
            <el-descriptions>
                <el-descriptions-item label="车辆停放时长">{{ parkDuration }}</el-descriptions-item>
            </el-descriptions>
            <el-descriptions>
                <el-descriptions-item label="结算费用">{{ amount }}</el-descriptions-item>
            </el-descriptions>
            <span style="color: #ff0000;font-size: 12px;">注：停放时长不满一小时按一小时计算</span>
            <span slot="footer" class="dialog-footer">
                <el-button @click="dialogVisible = false">取 消</el-button>
                <el-button type="primary" @click="handleConfirm">确 定</el-button>
            </span>
        </el-dialog>
    </div>
</template>

<script>
import { getSettlementList, postSettlementDeparture } from "@/api/access"
import { timeTransformation } from "@/utils/index"
import saveExcel from '@/utils/saveExcel.js'
export default {
    components: {},
    data() {
        return {
            // 查询条件
            form: {
                plateNumber: undefined, // 车牌号
                carNumber: undefined, // 车位号
                phone: undefined, // 手机号
                status: undefined, // 支付状态
                pageNum: 1,
                pageSize: 2
            },
            total: 0,
            tableData: [],
            dialogVisible: false,
            parkDuration: undefined,
            leavingTime: undefined,
            parkFee: undefined,
            amount: undefined,
            id: undefined,
            carNumber: undefined,
            exportLoading: false,
            multipleSelection: {},
            exportList: []
        };
    },
    created() {
        this.getList();
    },
    methods: {
        getList() {
            getSettlementList(this.form).then(({ data,total }) => {
                this.tableData = data;
                this.total = total;
                setTimeout(() => {
                    if (this.multipleSelection[this.form.pageNum]) {
                        this.multipleSelection[this.form.pageNum].forEach((row) => {
                            data.forEach((list) => {
                                if (list.id == row.id) {
                                    this.$refs.tableRef.toggleRowSelection(list, true);
                                }
                            });
                        });
                    }
                }, 0);
            })
        },
        // 确认离场
        handleDeparture(row) {
            this.dialogVisible = true;
            this.id = row.id;
            this.carNumber = row.carNumber;
            this.parkFee = row.chargeHour; // 当前车位停车费用
            let exittime = new Date(row.exittime).getTime() / 1000; // 将入场时间 转换成时间戳
            let newTime = new Date().getTime(); // 固定当前时间
            this.leavingTime = timeTransformation(newTime); // 获取当前时间并格式化
            let leavingTime = newTime / 1000;    // 将离场时间 转换成时间戳
            let differTime = leavingTime - exittime; // 相差时间 
            let getMin = parseFloat(differTime / 60); // 获得 多少分钟
            let parkDuration = getMin / 60; // 获取小时
            if(!Number.isInteger(parkDuration)) {
                this.parkDuration = parkDuration.toFixed(0);
                this.parkDuration = Number(this.parkDuration) + 1;
                return this.amount = this.parkFee * this.parkDuration;
            }
        },
        // 确认结算
        handleConfirm() {
            let body = {
                leavingTime: this.leavingTime,
                duration: this.parkDuration,
                amount: this.amount,
                id: this.id,
                carNumber: this.carNumber,
            }
            postSettlementDeparture(body).then(({status, message}) => {
                if(status === 0) {
                    this.dialogVisible = false;
                    this.msgSuccess(message);
                    this.getList();
                }
            })
        },
        // 重置
        cancel(formName) {
            this.$refs[formName].resetFields();
            this.getList();
        },
        exportExcel() {
            this.exportLoading = true;
            this.exportList = [];
            for (let key in this.multipleSelection) {
                this.multipleSelection[key].forEach(item => {
                    if (this.exportList.indexOf(item) == -1) {
                        this.exportList.push(item)
                    }
                })
            }
            if (this.exportList.length === 0) {
                this.exportLoading = false;
                this.msgWarning('请选择需要导出的车辆登记信息!');
            } else {
                const options = [{
                    key: '车牌号',
                    value: 'plateNumber'
                }, {
                    key: '车位号',
                    value: 'carNumber'
                }, {
                    key: '车主姓名',
                    value: 'ownerName'
                }, {
                    key: '车主电话',
                    value: 'phone'
                }, {
                    key: '每小时收费(元)',
                    value: 'chargeHour'
                }, {
                    key: '入场时间',
                    value: 'exittime'
                }, {
                    key: '离场时间',
                    value: 'leavingTime'
                },{
                    key: '停放时长(小时)',
                    value: 'duration'
                },{
                    key: '应收费金额(元)',
                    value: 'amount'
                }, {
                    key: '是否支付',
                    value: 'status'
                }]
                this.exportList.forEach(item => {
                    if (item.status == 1) item.status = '已支付'
                    else item.status = '未支付'
                });
                saveExcel(options, this.exportList, '车辆结算信息');
                this.exportLoading = false;
                this.multipleSelection = {};
                this.getList();
            }
        },
        // 获取分页多选框的数据
        handleSelectionAll(val) {
            this.multipleSelection[this.form.pageNum] = val;
        },
        handleSelection(val) {
            this.multipleSelection[this.form.pageNum] = val;
        }
    }
}

</script>
<style lang='scss' scoped></style>