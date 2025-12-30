# Tabby Payment Integration – Android (Clean Architecture)

This document provides a **complete, end-to-end guide** for integrating **Tabby Payment**
into an Android application using **Kotlin**, **Clean Architecture**, and **MVVM**.

All steps, code snippets, UI handling, checkout flow, result handling,
and localization messages are included.

---

## 📌 Tech Stack

- Kotlin
- Android SDK
- MVVM + Clean Architecture
- Retrofit
- Kotlin Coroutines & Flow
- ViewBinding
- Tabby Android SDK

---

## 1️⃣ Dependencies

Add Tabby SDK to `app/build.gradle`:

```gradle
dependencies {
    implementation("ai.tabby:tabby-android:1.1.13")
}
```

---

## 2️⃣ Configuration

### 2.1 Config File

Create `config.properties`:

```properties
tabbyPublicKey=pk_test_xxxxx
tabbySecretKey=sk_test_xxxxx
tabbyMerchantCode=ADD_MURCHENT_CODE
```

---

### 2.2 BuildConfig Setup

```gradle
android {
    buildTypes {
        debug {
            buildConfigField("String","TABBY_PK","\"${myConfigProperties['tabbyPublicKey']}\"")
            buildConfigField("String","TABBY_SK","\"${myConfigProperties['tabbySecretKey']}\"")
            buildConfigField("String","TABBY_MC","\"${myConfigProperties['tabbyMerchantCode']}\"")
        }
        release {
            buildConfigField("String","TABBY_PK","\"${myConfigProperties['tabbyPublicKey']}\"")
            buildConfigField("String","TABBY_SK","\"${myConfigProperties['tabbySecretKey']}\"")
            buildConfigField("String","TABBY_MC","\"${myConfigProperties['tabbyMerchantCode']}\"")
        }
    }
}
```

---

## 3️⃣ Data Models

```kotlin
data class TabbyResponse(
    @Json(name = "success") val success: Boolean?,
    @Json(name = "status_code") val statusCode: Int?,
    @Json(name = "session_status") val sessionStatus: String?,
    @Json(name = "configuration") val configuration: Configuration?
)

data class Configuration(
    @Json(name = "products") val products: Products?
)

data class Products(
    @Json(name = "installments") val installments: Installments?
)

data class Installments(
    @Json(name = "type") val type: String?,
    @Json(name = "is_available") val isAvailable: Boolean?,
    @Json(name = "rejection_reason") val rejectionReason: String?
)
```

---

## 4️⃣ API & Repository

### Retrofit API

```kotlin
@FormUrlEncoded
@POST("create-session")
suspend fun createTabbySession(
    @Field("order_id") offerId: String
): BaseResponse<TabbyResponse>
```

### Repository Interface

```kotlin
suspend fun createTabbySession(
    orderId: String
): Flow<DataState<BaseResponse<TabbyResponse>>>
```

### Repository Implementation

```kotlin
override suspend fun createTabbySession(orderId: String) =
    safeApiCall {
        ordersRemoteDataSource.createTabbySession(orderId)
    }
```

---

## 5️⃣ ViewModel

```kotlin
private val _tabbySessionListener =
    MutableStateFlow<DataState<BaseResponse<TabbyResponse>>>(DataState.Idle)

val tabbySessionListener = _tabbySessionListener.asStateFlow()

fun getTabbySession(orderId: String) {
    viewModelScope.launch {
        ordersRepository.createTabbySession(orderId).collectLatest {
            _tabbySessionListener.value = it
        }
    }
}
```

---

## 6️⃣ UI Integration

### Layout

```xml
<FrameLayout
    android:id="@+id/tabbyContainer"
    android:layout_width="match_parent"
    android:layout_height="wrap_content" />
```

### Widget Setup

```kotlin
tabbyWidget = TabbyInstallmentsWidget(requireContext()).apply {
    currency = Currency.SAR
    amount = BigDecimal(price)
}
binding.tabbyContainer.addView(tabbyWidget)
```

### Session Handling

```kotlin
when (response.data?.sessionStatus) {
    "created" -> activeTabby()
    "rejected" -> disableTabby(
        response.data?.configuration
            ?.products
            ?.installments
            ?.rejectionReason
    )
}
```

```kotlin
    private fun disableTabby(status: String?) {
        paymentTypesAdapter.tabbyIsDisabled = true
        paymentTypesAdapter.tabbyMsg = getString(
            when (status) {
                "order_amount_too_high" -> R.string.order_amount_too_high
                "order_amount_too_low" -> R.string.order_amount_too_low
                else -> R.string.tabby_reject_msg
            }
        )
        binding.tabbyContainer.toGone()
        getPaymentMethods()
    }

    private fun activeTabby() {
        paymentTypesAdapter.tabbyIsDisabled = false
        paymentTypesAdapter.tabbyMsg = null
        binding.tabbyContainer.toGone()
        getPaymentMethods()
    }
```
---

## 7️⃣ initTabby(orderId: String)

```kotlin
private fun initTabby(orderId: String) {
    val tabbyPayment = TabbyPayment(
        amount = BigDecimal(order.totalPrice),
        currency = Currency.SAR,
        description = "Tabby Order #$orderId",
        buyer = Buyer(
            email = user.email,
            phone = user.phone,
            name = user.name
        ),
        order = Order(
            refId = orderId,
            items = listOf(
                OrderItem(
                    quantity = 1,
                    refId = orderId,
                    title = order.title,
                    unitPrice = BigDecimal(order.totalPrice),
                    category = order.category
                )
            )
        )
    )

    lifecycleScope.launchWhenCreated {
        val session = TabbyFactory.tabby.createSession(
            merchantCode = BuildConfig.TABBY_MC,
            lang = Lang.EN,
            payment = tabbyPayment
        )

        session.availableProducts.forEach {
            val intent = TabbyFactory.tabby.createCheckoutIntent(it)
            checkoutContract.launch(intent)
        }
    }
}
```

---

## 8️⃣ Checkout Result Handling

```kotlin
private val checkoutContract =
    registerForActivityResult(ActivityResultContracts.StartActivityForResult()) {
        if (it.resultCode == Activity.RESULT_OK) {
            it.data?.tabbyResult?.let { result ->
                when (result.result) {
                    TabbyResult.Result.AUTHORIZED -> navigateToSuccess()
                    TabbyResult.Result.REJECTED -> showError(R.string.tabby_reject_msg)
                    TabbyResult.Result.CLOSED -> showError(R.string.tabby_closed_msg)
                    TabbyResult.Result.EXPIRED -> showError("expired")
                }
            }
        }
    }
```

---

## 9️⃣ String Resources

### Arabic (`values-ar/strings.xml`)

```xml
<string name="tabby_reject_msg">أسف، تابي غير قادرة على الموافقة على هذه العملية. الرجاء استخدام طريقة دفع أخرى</string>
<string name="order_amount_too_high">قيمة الطلب تفوق الحد الأقصى المسموح به حاليًا مع تابي</string>
<string name="order_amount_too_low">قيمة الطلب أقل من الحد الأدنى المطلوب لاستخدام خدمة تابي</string>
<string name="tabby_closed_msg">لقد ألغيت الدفعة. فضلاً حاول مجددًا</string>
```

### English (`values/strings.xml`)

```xml
<string name="tabby_reject_msg">Sorry, Tabby is unable to approve this purchase</string>
<string name="order_amount_too_high">This purchase is above your current spending limit</string>
<string name="order_amount_too_low">The purchase amount is below the minimum amount</string>
<string name="tabby_closed_msg">You aborted the payment</string>
```

---

## ✅ Final Notes

- Full Clean Architecture compliance
- No business logic in UI
- Production-ready
- Supports Arabic & English
- Safe for GitLab documentation
