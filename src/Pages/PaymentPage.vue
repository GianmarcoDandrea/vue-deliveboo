<script>
import { store } from '../store';
import axios from 'axios';
import { toast } from 'vue3-toastify';
import 'vue3-toastify/dist/index.css';
import PrintReceiptComponent from '../components/PrintReceiptComponent.vue';
export default {
    inject: ['providedMethod', 'providedAddToCart', 'providedRemoveFromCart', 'providedClearCart', 'providedSaveCartToLocalStorage', 'providedLoadCartFromLocalStorage'],
    name: "App",
    props: {
        selectedRestaurantId: String,
        selectedRestaurant: String,
    },
    components: {
        PrintReceiptComponent
    },
    data() {
        return {

            store,
            cart: [],
            customers_name: '',
            customers_phone_number: '',
            customers_address: '',
            customers_email: '',
            selectedRestaurantId: '',
            isOrderSuccessful: false,
            orderData: {},
        };
    },
    mounted() {
        this.initializeDropin();
        this.providedLoadCartFromLocalStorage();
        const storedCart = localStorage.getItem('cart');
        if (storedCart) {
            this.cart = JSON.parse(storedCart);
            console.log(this.cart);
        };

        // SALVO I DATI NEL FORM AFFINCHE VENGANO REINSERITI ALLA RICARICA DELLA PAGINA
        // const savedData = JSON.parse(localStorage.getItem('formData'));
        // if (savedData) {
        //     this.customers_name = savedData.customers_name || '';
        //     this.customers_phone_number = savedData.customers_phone_number || '';
        //     this.customers_address = savedData.customers_address || '';
        //     this.customers_email = savedData.customers_email || '';
        // }
        // this.$watch(
        //     () => ({
        //         customers_name: this.customers_name,
        //         customers_phone_number: this.customers_phone_number,
        //         customers_address: this.customers_address,
        //         customers_email: this.customers_email
        //     }),
        //     (newValue) => {
        //         localStorage.setItem('formData', JSON.stringify(newValue));
        //     },
        //     { deep: true }
        // );
    },
    computed: {
        totalCount() {
            return this.cart.reduce((total, item) => total + item.count, 0);
        },
        cartTotal() {
            const totalAmount = this.cart.reduce(
                (total, item) => total + parseFloat(item.price * item.count),
                0
            );
            return totalAmount.toFixed(2);
        },
        totalItemCount() {
            return this.cart.reduce((total, item) => total + item.count, 0);

        },
        orderRoute() {
            if (this.cart.length > 0 && this.cart[0].restaurant_id) {
                const restaurantId = this.cart[0].restaurant_id;
                return `${this.store.baseUrl}/api/restaurant/${restaurantId}/orders`;
            }
            return null;
        },
        isEmailValid() {
            
            const emailRegex = /\S+\S+\.\S+/;
            return emailRegex.test(this.customers_email);
        },
        isNameValid() {
        return this.customers_name.trim().length > 0;
        },
        isPhoneNumberValid() {
            const phoneNumberRegex = /^[\+]?[(]?[0-9]{3}[)]?[-\s\.]?[0-9]{3}[-\s\.]?[0-9]{4,6}$/im;
            return phoneNumberRegex.test(this.customers_phone_number);
        },
        isAddressValid() {
            return this.customers_address.trim().length > 0;
        },
        isFormValid() {
            return this.isNameValid && this.isPhoneNumberValid && this.isAddressValid && this.isEmailValid;
        },

    },
    methods: {
        notifySuccess() {
            toast("Payment done, order processed!", {
                autoClose: 5000,
                type: "success"
            });
        },
        notifyError(message) {
            toast('Pagamento non riuscito, inserisci un altro metodo di pagamento o contatta il tuo istituto bancario', {
                autoClose: 5000,
                type: "error"
            });
        },
         notifyValidation(message) {
            toast('Inserisci i tuoi dati, per favore', {
                autoClose: 5000,
                type: "error"
            });
        },
        notifyErrorServer(){
            toast('Qualcosa è andato storto, si prega di riprovare più tardi :(',{
                autoClose: 5000,
                type:'error'
            })
        },
        clearCart() {
            this.cart = []; 
        },
        async initializeDropin() {
            try {
                // token di autorizzazione 
                const response = await fetch(`${this.store.baseUrl}/api/payment/token`);
                const { token } = await response.json();
                console.log('Authorization Token:', token);
                // crea istanza dropin con token ricevuto
                braintree.dropin.create({
                    authorization: token,
                    container: '#dropin-container',
                }, (error, dropinInstance) => {
                    if (error) {
                        console.error('Errore nella creazione del Drop-in:', error);
                        return;
                    }
                    this.dropinInstance = dropinInstance;
                });
            } catch (error) {
                console.error('Errore nel recupero del token del client:', error);
            }
        },

        // processo di checkout
        async submitCheckout() {
                if (!this.isFormValid) {
                this.notifyValidation('Please make sure all fields are correctly filled.');
                return;
                }


            // prende il metodo di pagamento selezionato dall'utente
            this.dropinInstance.requestPaymentMethod(async (error, payload) => {
                if (error) {
                    console.error('Errore nella richiesta del metodo di pagamento:', error);
                    return;
                }

                // prepara i dati di pagamento per l'invio al server
                const paymentData = {
                    payment_method_nonce: payload.nonce,
                    amount: this.cartTotal
                };
                console.log('Payload:', payload);

                try {
                    //  richiesta di pagamento al server
                    const paymentResponse = await axios.post(`${this.store.baseUrl}/api/payment/checkout`, paymentData);
                    console.log('Risposta del pagamento:', paymentResponse);

                    if (paymentResponse.data.success) {
                        //prepara i dati dell'ordine per l'invio al backend
                        const paymentResponseData = paymentResponse.data;

                        //ID della transazione
                        const transactionId = paymentResponseData.transaction.id;
                        console.log(transactionId)
                        const orderData = this.prepareOrderData(transactionId);
                        console.log('Dati ordine:', orderData);


                        try{
                            // invia l'ordine al backend
                            const orderResponse = await this.submitOrderToBackend(orderData);
                            console.log(orderResponse)


                                if (orderResponse.status === 201) {
                                    //estrai messaggio da response data 
                                    const responseData = orderResponse.data;
                                    const message = responseData.message;
                                    console.log('Messaggio di conferma dell\'ordine:', message);
                                    // operazioni post-checkout se checkout ok
                                    this.orderData = this.prepareOrderData();
                                    this.providedClearCart();
                                    this.providedSaveCartToLocalStorage();
                                    console.log('Ordine effettuato e carrello svuotato');
                                    //resetta dati cliente, prende orderdata x ricevuta, invia toast
                                    this.customers_name = '';
                                    this.customers_phone_number = '';
                                    this.customers_address = '';
                                    this.clearCart();

                                    this.isOrderSuccessful = true;
                                    this.notifySuccess();
                                    this.cart = [];
                                }else{
                                    // se ci sono errori con status risposta 
                                    console.error('Errore nell\'invio dell\'ordine. Status:', orderResponse.status);
                                    this.notifyErrorServer('Errore nell\'invio dell\'ordine. ');
                                }
                        } catch (error) {
                            this.notifyErrorServer();
                            console.error('Errore nell\'invio dell\'ordine:', error);
                            //stop se ci sono problemi
                            return; 
                        }


                    
                    } else {
                        this.notifyError();
                        console.error('Pagamento non riuscito, contatta il tuo istituto bancario');
                    }
                } catch (error) {
                    this.notifyErrorServer();
                    console.error('Errore nel processo di pagamento:', error);
                }
            });
        },

        // preparare dati x invio ordine
        prepareOrderData(transactionId) {
            return {
                transactionId: transactionId,
                customers_name: this.customers_name,
                customers_phone_number: this.customers_phone_number,
                customers_address: this.customers_address,
                customers_email: this.customers_email,
                food_items: this.cart.map(item => ({
                    id: item.id,
                    quantity: item.count,
                    price: item.price
                })),
            }
        },

        // inviare l'ordnie al backend con rotta dinamica a seconda del restaurant id 
        async submitOrderToBackend(orderData) {
            const route = this.orderRoute;
            console.log('Rotta per l\'invio dell\'ordine:', this.orderRoute)
            if (!route) {
                console.error('Rotta non definita');
                return;
            }

            try {
                // richiesta di invio dell'ordine al backend
                const response = await axios.post(route, orderData);
                console.log('Risposta dell\'invio dell\'ordine:', response);
                return response;
            } catch (error) {
                this.notifyErrorServer();
                console.error('Errore nell\'invio dell\'ordine:', error);
                return { error: true, errorMessage: error.message };
            }
        },
        goBack() {
            console.log('click!');
            console.log("Tornare al ristorante", this.cart[0].restaurantSlug, "Nome:", this.cart[0].restaurantName);
            //se il carrello è pieno e c'è restaurant slug
           if (this.cart.length > 0 && this.cart[0].restaurantSlug) {
                this.$router.push({ name: 'restaurant-details-page', params: { slug: this.cart[0].restaurantSlug } });
                //altrimenti, torna indietro
            }else{
                this.$router.go(-1);
            }
        },
        removeFromCart(index) {
            this.cart.splice(index, 1);
            this.saveCartToLocalStorage();
            window.location.reload();
        },
        saveCartToLocalStorage() {
            localStorage.setItem('cart', JSON.stringify(this.cart));
        },
    },
}
</script>


<template>
    <section class="h-100 p-3">
        <div class="container p-3 h-100" style="background-color: rgb(47 38 38)">
            <div class="row d-flex justify-content-center align-items-center h-100">
                <div class="col">
                    <div class="card">
                        <div class="card-body p-4 order">

                            <div class="row">

                                <div class="col-lg-7">
                                    <h5 class="mb-4">

                                        <!-- * MENU ROUTE -->
                                        <!-- TODO: aggiungere la rotta che riporti al menu -->
                                        <ul id="breadcrumb" class="breadcrumbs-container container-fluid d-flex">
                                            <li><router-link :to="{ name: 'home' }"> <i class="fa-solid fa-house"> </i>
                                                </router-link></li>
                                                <li><a @click="goBack()"><i class="fa-solid fa-utensils"></i> 
                                                    <span v-if="cart.length > 0 && cart[0].restaurantSlug">
                                                        <!-- mostra nome ristorante preso da primo item in cart -->
                                                            {{ cart[0].restaurantName }}
                                                        </span>
                                                        <!-- se cart è vuoto, go back -->
                                                    <span v-else>
                                                            Back
                                                    </span></a></li>
                                            <li><a disabled><i class="fa-solid fa-bag-shopping"></i>
                                                    <span class="ms-1">
                                                        Checkout
                                                    </span></a>
                                            </li>
                                        </ul>
                                    </h5>

                                    <hr>

                                    <div class="d-flex justify-content-between align-items-center mb-4 .ms_card">
                                        <div>
                                            <h3 class="mb-2 title">Order Summary</h3>

                                            <!-- * CART ITEMS COUNT -->
                                            <!-- TODO: aggiungere il count degli items presenti nel ordine -->

                                            <p class="mb-0 mt-2 ms-2">You have <strong>{{ totalCount }}</strong> items

                                                in your cart</p>
                                        </div>
                                    </div>

                                    <hr>

                                    <!-- * CART ITEMS SUMMARY CARD  -->
                                    <div class="card mb-3 mt-4 " v-for="item in store.cart">
                                        <div class="card-body dish text-white ">
                                            <div class="d-flex justify-content-between ms_card">
                                                <div class="d-flex flex-row align-items-center ms_card">
                                                    <div>
                                                        <!-- TODO: aggiungere Immagini piatti-->
                                                        
                                                    </div>
                                                    <div class="ms-3">
                                                        <h5>{{ item.name }}</h5>
                                                        <p class="small mb-0">{{ item.description }}</p>
                                                    </div>
                                                </div>
                                                <div
                                                    class="d-flex flex-row align-items-center justify-content-center text-center ms_card">
                                                    <div style="width: 50px;">
                                                        <h5 class="fw-normal mb-0"><strong>QTY</strong></h5>
                                                        <p> {{ item.count }} </p>
                                                    </div>
                                                    <div style="width: 80px;">
                                                        <h5 class="mb-0"><strong>PRICE</strong></h5>
                                                        <p> {{ item.price }} €</p>
                                                    </div>
                                                    <!-- <router-link to="/payments" class="btn fw-bold"> -->
                                                    <a href="#" style="color: #d1d1d1;" @click="removeFromCart($index)">
                                                        <i class="fas fa-trash-alt ms-4">
                                                        </i>
                                                    </a>
                                                    <!-- </router-link> -->


                                                </div>
                                            </div>
                                        </div>
                                    </div>
                                    <PrintReceiptComponent :isVisible="isOrderSuccessful" :orderData="orderData"
                                        @close="isOrderSuccessful = false"></PrintReceiptComponent>

                                </div>

                                <!-- * CHECKOUT  -->
                                <form class="col-lg-5">
                                    <div class="card bg-warning rounded-3 text-white p-2">
                                        <div class="card-body check">
                                            <div class="d-flex justify-content-between align-items-center mb-4">
                                                <h3 class="mb-0">Checkout</h3>

                                            </div>

                                            <!-- * CARD TYPES ICONS -->
                                            <h4 class="small mb-2">Card type</h4>
                                            <a href="#!" type="submit" class="text-white"><i
                                                    class="fab fa-cc-mastercard fa-2x me-2"></i></a>
                                            <a href="#!" type="submit" class="text-white"><i
                                                    class="fab fa-cc-visa fa-2x me-2"></i></a>
                                            <a href="#!" type="submit" class="text-white"><i
                                                    class="fab fa-cc-amex fa-2x me-2"></i></a>
                                            <a href="#!" type="submit" class="text-white"><i
                                                    class="fab fa-cc-paypal fa-2x"></i></a>


                                            <div class="mt-4" @submit.prevent="submitCheckout">
                                                <!-- * CUSTOMER NAME INPUT-->
                                                <div class="form-outline form-white mb-2">
                                                    <input type="text" id="customers_name" v-model="customers_name"
                                                        class="form-control form-control-lg" siez="17"
                                                        placeholder="Your Name" required />
                                                    <label class="form-label ms-2 mt-1" for="customers_name">First name and
                                                        last Name</label>
                                                        <small v-if="!customers_name" class="text-danger">Please enter your name.</small>
                                                </div>

                                                <!-- *NUMBER INPUT-->
                                                <div class="form-outline form-white mb-2">
                                                    <input type="tel" id="customers_phone_number"
                                                        v-model="customers_phone_number"
                                                        class="form-control form-control-lg" size="17"
                                                        placeholder="Phone number" minlength="10" maxlength="15" required />
                                                        <small v-if="!customers_phone_number" class="text-danger">Please enter your phone number.</small>
                                                    <label class="form-label ms-2 mt-1" for="customers_phone_number">Phone
                                                        Number</label>
                                                </div>

                                                <!-- * ADDRESS INPUT-->
                                                <div class="form-outline form-white mb-2">
                                                    <input type="text" id="customers_address" v-model="customers_address"
                                                        class="form-control form-control-lg" placeholder="Your address"
                                                        size="17" required/>
                                                    <label class="form-label ms-2 mt-1"
                                                        for="customers_address">Address</label>
                                                </div>


                                                <!-- *EMAIL INPUT-->

                                                <div class="form-outline form-white mb-2">
                                                    <input type="email" id="customers_email" v-model="customers_email"
                                                        class="form-control form-control-lg" placeholder="email" size="7" required />
                                                        <small v-if="!customers_email" class="text-danger">Please enter your email.</small>
                                                        <small v-else-if="!isEmailValid" class="text-danger">Please enter a valid email address.</small>
                                                    <label class="form-label ms-2 mt-1" for="customers_email">Email</label>
                                                </div>

                                            </div>

                                            <hr class="mt-4">

                                            <!-- * CHECKOUT BUTTON -->

                                            <!-- Drop-in UI container -->
                                            <div id="dropin-container">

                                            </div>
                                            

                                            <hr class="mb-4 mt-5">

                                            <!-- * TOTAL PRICE -->
                                            <div class="d-flex justify-content-between mt-4">
                                                <h5 class="mb-2">Total (Incl. taxes)</h5>
                                                <h5 class="mb-2">{{ cartTotal }} €</h5>
                                            </div>

                                            <hr class="mb-4">

                                            <button type="button" id="submit-button" @click="submitCheckout"
                                                class="btn btn-warning btn-block btn-lg mt-3 d-flex align-items-end" :disabled="!isFormValid" >
                                                <span>Checkout</span>
                                            </button>

                                        </div>
                                    </div>

                                </form>



                            </div>

                        </div>
                    </div>
                </div>
            </div>
        </div>
    </section>
</template>


<style lang="scss">

@media screen and (max-width: 596px){
.ms_card {
    display: flex;
    flex-direction: column;
    flex-wrap: wrap;
    justify-content: center;
    align-items: center;
    gap: 1rem;
    text-align: center;
}
}
.order {
    background-color: rgb(197, 170, 106) !important;
}

.check {
    background-color: rgb(47 38 38) !important;
}

.title {
    color: white;
    background-color: rgb(47 38 38);
    padding: 10px;
    border-radius: 1rem;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.6);
}

.continue {
    background-color: rgb(255 193 7);
    padding: 10px;
    border-radius: 1rem;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.6);
}

.fa-trash-alt {
    &:hover {
        color: red;
    }
}

.dish {
    background-color: rgb(47 38 38);
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.6);
    border-color: black;
}

// breadcrumb
$yellow: rgba(47, 38, 38, 0.3);
$yellow-darken: darken(rgba(47, 38, 38, 0.4), 20%);
$hover-color: darken(rgba(47, 38, 38, 0.5), 20%);
$active-color: #f2c802;


#breadcrumb {
    list-style: none;
    display: block;

    .icon {
        font-size: 14px;
    }

    li {
        float: left;
        padding: 10px 0 10px 0px;
        cursor: pointer;

        a {
            color: #fff;
            display: block;
            background: $yellow;
            text-decoration: none;
            position: relative;
            height: 20.80px;
            line-height: 20.80px;
            padding: 0 10px 0 2px;
            text-align: center;
            font-size: 0.8rem;
            margin-right: 8px;
        }

        &:nth-child(even) {
            a {
                background-color: $yellow;

                &:after {
                    border-left-color: $yellow;
                }
            }
        }

        &:first-child {
            a {
                padding-left: 15px;
                border-radius: 4px 0 0 4px;

                &:before {
                    border: none;
                }
            }
        }

        &:last-child {
            a {
                padding-right: 15px;
                padding-left: 15px;
                border-radius: 4px;
                background-color: $yellow-darken;

                &:after {
                    border: none;
                }
            }
        }

        a {
            border-radius: 4px 0 0 4px;
            padding-left: 15px;

            // &:before,
            &:after {
                content: "";
                position: absolute;
                top: 0;
                border: 0 solid $yellow;
                border-width: 10.4px 4.8px;
                width: 0;
                height: 0;
                display: block;
            }

            // &:before{
            //     left: -9%;
            //     border-left-color:transparent;
            //     border-right-color:$yellow;
            // }
            &:after {
                left: 100%;
                border-color: transparent;
                border-left-color: $yellow;
            }

            &:hover {
                background-color: $hover-color;

                //   &:before{
                //     border-color:$hover-color;
                //     border-left-color:transparent;
                //    }
                &:after {
                    border-left-color: $hover-color;
                }
            }

            &:active {
                background-color: $active-color;

                //   &:before{
                //     border-color:$active-color;
                //     border-left-color:transparent;
                //    }
                &:after {
                    border-left-color: $active-color;
                }
            }
        }
    }
}
</style>