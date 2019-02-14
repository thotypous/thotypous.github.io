---
layout: post
title: Fixing an airline app bug for fun and profit
categories: reversing
tags: android reversing patching locale icu
---

Azul Linhas Aéreas is a Brazilian airline which offers cheaper tickets for customers buying from their mobile app. If the same tickets are bought from their website, an extra "convenience fee" is charged. Unfortunately, the mobile app has had a bug for several months which causes it to close itself when the user fills out the payment method. In this article, we patch the app's bytecode to fix the bug.

<!--more-->

# Problem statement

When booking a flight using their website, Azul charges a "convenience fee" of about 60 BRL per passenger. It is funny how the "convenience fee" is often more expensive than the embarkation fee charged by the airports.

<img src="/postdata/2019-02-13/convenience-fee.png" alt="Booking a flight in Azul's website" width="284" />

Luckily enough, flights booked using their mobile app are exempt from the "convenience fee".

<img src="/postdata/2019-02-13/no-fee.png" alt="Booking the same flight in Azul's mobile app" width="284" />

However, you cannot conclude the booking since the app crashes just after you select the payment method.

<img src="/postdata/2019-02-13/payment-bug.png" alt="The app crashes after selecting the payment method" width="284" />


# Debugging the issue

Since the app is suddenly being closed, the first hypothesis which comes at mind is that some uncaught exception is being fired and the app has no crash reporting mechanism (or has a silent one).

In order to test this hypothesis, we launch `adb logcat` and try to book a flight. The app crashes and we get the following stack trace.

```
java.lang.NumberFormatException: For input string: " 5519"
	at java.lang.Long.parseLong(Long.java:594)
	at java.lang.Long.valueOf(Long.java:808)
	at br.com.voeazul.controller.comprapassagem.CompraPassagemController.removerSeparadoresDeValor(CompraPassagemController.java:2982)
	at br.com.voeazul.controller.comprapassagem.CompraPassagemController.actionGetPaymentInstallmentInfo(CompraPassagemController.java:1922)
	at br.com.voeazul.controller.comprapassagem.CompraPassagemController.actionApplyPromotionCode(CompraPassagemController.java:1830)
	at br.com.voeazul.fragment.comprapassagem.CompraPassagemPagamentoFragment$2$1.onItemSelected(CompraPassagemPagamentoFragment.java:240)
	at android.widget.AdapterView.fireOnSelected(AdapterView.java:944)
	at android.widget.AdapterView.dispatchOnItemSelected(AdapterView.java:933)
	at android.widget.AdapterView.access$300(AdapterView.java:53)
	at android.widget.AdapterView$SelectionNotifier.run(AdapterView.java:898)
	at android.os.Handler.handleCallback(Handler.java:873)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loop(Looper.java:193)
	at android.app.ActivityThread.main(ActivityThread.java:6669)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:493)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:858)
```

Nice, so it seems like the app is trying to convert the amount due (in cents) to long. However, there are some leading spaces in the string, causing the conversion to fail. Did they forget to trim the string?

We retrieve the apk from the device and try to open it using [Bytecode Viewer](https://github.com/Konloch/bytecode-viewer). No luck – it gives up because it could not decode some resource in `resources.arsc`. However, this is not a problem, since we only intend to understand (and modify) the bytecode, thus we extract `classes.dex` and open it.

Let's decompile the culprit method mentioned in the stack trace:

{% highlight java %}
public long removerSeparadoresDeValor(final BigDecimal bigDecimal) {
    final DecimalFormat decimalFormat = (DecimalFormat)NumberFormat.getCurrencyInstance(new Locale("pt", "BR"));
    decimalFormat.setMaximumFractionDigits(2);
    final DecimalFormatSymbols decimalFormatSymbols = decimalFormat.getDecimalFormatSymbols();
    decimalFormatSymbols.setCurrencySymbol("");
    decimalFormat.setDecimalFormatSymbols(decimalFormatSymbols);
    return Long.valueOf(decimalFormat.format(bigDecimal).replace(".", "").replace(",", "").trim());
}
{% endhighlight %}

How amusing! They trim the string. So why is there leading whitespace?

After opening logcat's output in a hex editor, we realize the whitespace char is composed by the bytes `c2 a0` (UTF-8), which correspond to Unicode code point [00A0 (no-break space)](http://www.fileformat.info/info/unicode/char/00a0/index.htm). The `trim` method does not seem to strip it.


# Quick and dirty fix

The decompiler is rarely able to generate well-formed enough Java to be recompiled into a class. Therefore, it is easier to work directly in bytecode, by changing Bytecode Viewer's pane mode to editable Smali/DEX.

We look for the `removerSeparadoresDeValor` method and insert the following instructions just after the last call to `replace`.

```
move-result-object v0

const-string v1, "\u00a0"

const-string v2, ""

invoke-virtual {v0, v1, v2}, Ljava/lang/String;->replace(Ljava/lang/CharSequence;Ljava/lang/CharSequence;)Ljava/lang/String;
```

This causes `.replace("\u00a0", "")` to be called before `.trim()`.

Bytecode Viewer is able to save the modified code back to the `classes.dex` file (by calling Smali under the hood). However, we need to zip it back inside the apk manually, since we were not able to open the apk directly in Bytecode Viewer.

Then, we sign the apk with the debug key:

{% highlight bash %}
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore $HOME/.android/debug.keystore azul.apk androiddebugkey   # password: android
{% endhighlight %}

Finally, we just need to uninstall the original app and install our modified apk.


# Results

I was able to book a flight for my wife and me and paid only 110.38 BRL (embarkation fee for two passengers), whereas, if we had booked from the website, we would need to pay 229.98 BRL.


# Discussion

So far so good, after all, we fixed the bug and saved some money. But, as a scientist, you might be asking yourself some questions.

## What is the root cause?

The first interesting fact is that the issue does not happen if the `removerSeparadoresDeValor` is run in OpenJDK. It turns out OpenJDK has a [pure Java](https://github.com/unofficial-openjdk/openjdk/blob/2dfd47837ce6b900a0b3c9d6f09fbe1f62176c29/src/java.base/share/classes/java/text/DecimalFormat.java#L818-L842) implementation of `DecimalFormat`. Looking at the [locale data](https://github.com/unofficial-openjdk/openjdk/blame/ba116d532f477a597cbebfd5551b61a408136618/src/jdk.localedata/share/classes/sun/text/resources/ext/FormatData_pt_BR.java#L54) it uses, we can see it inserts an ordinary space (`\u0020`) after the currency symbol (`\u00A4`).

On the other hand, Android's implementation of `DecimalFormat` is a [wrapper for icu4c](https://github.com/AndroidSDKSources/android-sdk-sources-for-api-level-28/blob/809dd8b7822332cdbc68939d777eb70a6951908a/java/text/DecimalFormat.java#L705-L714). Looking at the [locale data](https://github.com/unicode-org/icu/blame/5c8960e59e1f71b102fa6ecc4ed4dbd36019d7ce/icu4c/source/data/locales/pt.txt#L35) it uses, we can see it inserts a no-break space (`\u00A0`) after the currency symbol (`\u00A4`).

## Since when the bug started to manifest itself?

The [locale data](https://github.com/unicode-org/icu/blame/5c8960e59e1f71b102fa6ecc4ed4dbd36019d7ce/icu4c/source/data/locales/pt.txt#L35) responsible for currency format was changed for the last time on 26 Sep 2017. [Before the change](https://github.com/unicode-org/icu/blob/ad11ee3a709e422d6e71ceb1e2f5ef45a5d3af12/icu4c/source/data/locales/pt.txt#L32), there was no whitespace at all between the currency symbol and the numeric value.

However, an interesting fact is that current icu4c code automatically [inserts a no-break space](https://github.com/unicode-org/icu/blame/b098078ab8a73e2f65c67accdb0bee4fd170044a/icu4c/source/data/curr/root.txt#L203) between the currency symbol and the numeric value even if it is not present in the pattern specified in locale data. Since when does this happen? The [new formatter](https://unicode-org.atlassian.net/browse/ICU-13634) responsible for [adding the spacing](https://github.com/unicode-org/icu/blame/249e03ccd6909a7528afdd0cbf8814a0ae9e3801/icu4c/source/i18n/number_modifiers.cpp#L377-L395) was first commited on 26 Sep 2017, and the [unit tests](https://github.com/unicode-org/icu/blame/b6074fe044f675f63475d25d9a5c149cf9f1dd1a/icu4c/source/test/intltest/numrgts.cpp#L2391-L2394) were fixed on 17 Apr 2018. So it came only after the change in locale data.

Since when does Android ship this locale data? Android 9 [has it](https://android.googlesource.com/platform/external/icu/+/android-9.0.0_r1/icu4c/source/data/locales/pt.txt#33) since the first release. Android 8 [does not have it](https://android.googlesource.com/platform/external/icu/+/android-8.1.0_r61/icu4c/source/data/locales/pt.txt#33) even in recent releases.

## How come the app developers did not notice it for several months?

Obviously they did not test the app in Android 9. But was it the only way they could get to know about the bug? Do they collect crash reports?

It is funny that the [only uncaught exception handler](https://github.com/TimotheeJeannin/AppRate/blob/master/AppRate/src/com/tjeannin/apprate/ExceptionHandler.java) the app installs is related to app rating. Yes, it serves the only purpose of avoiding to ask users to rate the app if the app crashed at least once for them. Of course, these users would not give many stars.


# Conclusions

Perhaps it was easier to just launch an Android 8 emulator and book the flight from there. But, how could we know?

We strongly recommend Azul Linhas Aéreas to:

 * Reimplement the `removerSeparadoresDeValor` method adopting a more reliable implementation (our quick and dirty fix is not an example!), avoiding dependence on locale data to work correctly.

 * Start collecting crash reports and deal with them accordingly.
