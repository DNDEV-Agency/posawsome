<template>
    <div>
        <v-data-table :headers="invoices_headers" :items="outstanding_invoices" item-key="name" class="elevation-1 mt-0"
            show-select v-model="selected_invoices" :loading="invoices_loading" checkbox-color="primary"
            @item-selected="onInvoiceSelected">
            <template v-slot:item.grand_total="{ item }">
                {{ currencySymbol(item.currency) }}
                {{ formtCurrency(item.grand_total) }}
            </template>
            <template v-slot:item.outstanding_amount="{ item }">
                <span class="primary--text">{{ currencySymbol(item.currency) }}
                    {{ formtCurrency(item.outstanding_amount) }}</span>
            </template>
        </v-data-table>
    </div>
</template>

<script>
import { evntBus } from "../../bus";
import format from "../../format";
export default {
    mixins: [format],
    props: ["customer_name", "pos_profile", "pos_profile_search"],
    data: function () {
        return {
            invoices_headers: [
                {
                    text: __("Invoice"),
                    align: "start",
                    sortable: true,
                    value: "name",
                },
                {
                    text: __("Customer"),
                    align: "start",
                    sortable: true,
                    value: "customer_name",
                },
                {
                    text: __("Date"),
                    align: "start",
                    sortable: true,
                    value: "posting_date",
                },
                {
                    text: __("Due Date"),
                    align: "start",
                    sortable: true,
                    value: "due_date",
                },
                {
                    text: __("Total"),
                    align: "end",
                    sortable: true,
                    value: "grand_total",
                },
                {
                    text: __("Outstanding"),
                    align: "end",
                    sortable: true,
                    value: "outstanding_amount",
                },
            ],
            outstanding_invoices: [],
            invoices_loading: false,
            selected_invoices: [],
        }
    },
    mounted() {
        this.$nextTick(function () {
            this.check_opening_entry();
            evntBus.$on("update_customer", (customer_name) => {
                this.customer_name = customer_name;
                if (!this.customer_name) {
                    this.clear_all();
                }
                if (this.company) {
                    this.get_outstanding_invoices();
                }
            });
            evntBus.$on("refresh_outstanding_invoices", () => {
                this.clear_all();
                this.get_outstanding_invoices();
            });
        });
    },
    created() {
        const vm = this;
        evntBus.$on('refresh-invoices', () => {
            vm.getOutstandingInvoices();
        });
    },
    methods: {
        check_opening_entry() {
            return frappe
                .call("posawesome.posawesome.api.posapp.check_opening_shift", {
                    user: frappe.session.user,
                })
                .then((r) => {
                    if (r.message) {
                        this.pos_profile = r.message.pos_profile;
                        this.pos_opening_shift = r.message.pos_opening_shift;
                        this.company = r.message.company.name;
                        evntBus.$emit("payments_register_pos_profile", r.message);
                        evntBus.$emit("set_company", r.message.company);
                        this.pos_profile_search = r.message.pos_profile.name;
                        this.payment_methods_list = [];
                        this.pos_profile.payments.forEach((element) => {
                            this.payment_methods_list.push(element.mode_of_payment);
                        });
                        this.get_outstanding_invoices();
                    }
                });
        },
        onInvoiceSelected(event) {
            evntBus.$emit("set_customer", event.item.customer);
            // setTimeout(() => {
            //     console.log("selected_invoices", this.selected_invoices.length);
            //     console.log("total_selected_invoices", this.total_selected_invoices);
            // }, 200);
            // evntBus.$emit("set_selected_invoices", {
            //     selected_invoices: this.selected_invoices,
            //     total_selected_invoices: this.total_selected_invoices,
            // });
        },
        get_outstanding_invoices() {
            this.invoices_loading = true;
            return frappe
                .call(
                    "posawesome.posawesome.api.payment_entry.get_outstanding_invoices",
                    {
                        customer: this.customer_name,
                        company: this.company,
                        currency: this.pos_profile.currency,
                        // pos_profile_name: this.pos_profile_search,
                    }
                )
                .then((r) => {
                    if (r.message) {
                        console.log("outstanding_invoices", r.message);
                        this.outstanding_invoices = r.message;
                        this.invoices_loading = false;
                    }
                });
        },
        clear_all() {
            this.outstanding_invoices = [];
            this.unallocated_payments = [];
            this.selected_invoices = [];
            this.total_selected_invoices = 0;
        },
    },
    computed: {
        total_selected_invoices() {
            return this.selected_invoices.reduce(
                (acc, cur) => acc + flt(cur.outstanding_amount),
                0
            );
        },
    },
    watch: {
        selected_invoices: function (val) {
            evntBus.$emit("set_selected_invoices", {
                selected_invoices: this.selected_invoices,
                total_selected_invoices: this.total_selected_invoices,
            });
        },
    },
}
</script>
