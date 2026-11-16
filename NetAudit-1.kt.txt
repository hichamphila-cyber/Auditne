package com.netaudit

import android.content.Context
import android.net.wifi.WifiManager
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.async
import kotlinx.coroutines.awaitAll
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.sync.Semaphore
import kotlinx.coroutines.sync.withPermit
import kotlinx.coroutines.withContext
import java.net.InetAddress
import java.net.InetSocketAddress
import java.net.Socket

enum class Severity { OK, INFO, WARN, HIGH }

data class Finding(val title: String, val detail: String, val severity: Severity)

data class Report(val score: Int, val findings: List<Finding>)

/**
 * NetAudit V1 — فحص دفاعي للشبكة التي يتصل بها الجهاز فقط.
 * لا يحاول كسر كلمات المرور ولا استغلال ثغرات ولا قطع اتصال أجهزة.
 */
@Suppress("DEPRECATION")
object NetAudit {

    private val PORTS = mapOf(
        21 to "FTP", 22 to "SSH", 23 to "Telnet", 80 to "HTTP", 443 to "HTTPS",
        445 to "SMB", 3389 to "RDP", 5555 to "ADB", 7547 to "TR-069", 8080 to "HTTP-alt"
    )

    suspend fun run(ctx: Context): Report {
        val app = ctx.applicationContext
        val findings = mutableListOf<Finding>()

        findings += wifiFinding(app)

        val gw = gatewayIp(app)
        if (gw == null) {
            findings += Finding("تعذّر تحديد الراوتر", "تأكد من اتصالك بشبكة Wi-Fi ثم أعد الفحص.", Severity.INFO)
        } else {
            findings += routerFindings(gw)
        }

        findings += devicesFinding(app)

        val penalty = findings.map {
            when (it.severity) {
                Severity.HIGH -> 25
                Severity.WARN -> 10
                else -> 0
            }
        }.sum()
        return Report((100 - penalty).coerceAtLeast(0), findings)
    }

    // ---------- Wi-Fi ----------

    private fun wifiFinding(ctx: Context): Finding {
        val wm = ctx.getSystemService(Context.WIFI_SERVICE) as WifiManager
        val caps: String? = try {
            val bssid = wm.connectionInfo?.bssid
            wm.scanResults.firstOrNull { it.BSSID == bssid }?.capabilities
        } catch (e: SecurityException) {
            null
        }
        return when {
            caps == null -> Finding(
                "تشفير الواي فاي",
                "تعذّر تحديد نوع التشفير. امنح التطبيق إذن الموقع وفعّل خدمة الموقع.",
                Severity.INFO
            )
            "SAE" in caps -> Finding("تشفير الواي فاي: WPA3", "أفضل مستوى حماية متاح.", Severity.OK)
            "[WPA-" in caps -> Finding(
                "WPA قديم مفعّل",
                "الشبكة تقبل WPA الأول الضعيف. اضبط الراوتر على WPA2 أو WPA3 فقط.",
                Severity.WARN
            )
            "WPA2" in caps -> Finding(
                "تشفير الواي فاي: WPA2",
                "جيد. يُفضَّل الانتقال إلى WPA3 إن كان الراوتر يدعمه.",
                Severity.OK
            )
            "WEP" in caps -> Finding("تشفير WEP", "تشفير مكسور. غيّره فورًا إلى WPA2/WPA3.", Severity.HIGH)
            else -> Finding("شبكة مفتوحة بلا تشفير", "أي شخص قريب يستطيع التنصّت. فعّل WPA2/WPA3.", Severity.HIGH)
        }
    }

    // ---------- الراوتر ----------

    private suspend fun routerFindings(gw: String): List<Finding> {
        val open = coroutineScope {
            PORTS.keys.map { p -> async(Dispatchers.IO) { if (isOpen(gw, p)) p else null } }
                .awaitAll().filterNotNull().sorted()
        }
        val out = mutableListOf<Finding>()

        if (23 in open) out += Finding("Telnet مفتوح على الراوتر", "بروتوكول غير مشفّر. عطّله من إعدادات الراوتر.", Severity.HIGH)
        if (5555 in open) out += Finding("منفذ ADB مفتوح", "يسمح بالتحكم عن بُعد في أجهزة أندرويد. أغلقه.", Severity.HIGH)
        if (21 in open) out += Finding("FTP مفتوح", "ينقل البيانات وكلمات المرور دون تشفير. عطّله أو استبدله بـ SFTP.", Severity.WARN)
        if (445 in open) out += Finding("SMB مفتوح على الراوتر", "تأكد أنه غير مكشوف خارج الشبكة المحلية.", Severity.WARN)
        if (3389 in open) out += Finding("RDP مفتوح", "تأكد أنه ضروري ومحمي بكلمة مرور قوية.", Severity.WARN)
        if (7547 in open) out += Finding("TR-069 مفتوح", "واجهة إدارة المزوّد. يجب ألا تكون متاحة داخل الشبكة المحلية.", Severity.WARN)
        if (80 in open && 443 !in open) out += Finding(
            "لوحة إدارة الراوتر عبر HTTP فقط",
            "بيانات الدخول تُرسل دون تشفير. فعّل HTTPS إن أمكن.",
            Severity.WARN
        )
        if (out.isEmpty()) out += Finding("لا منافذ خطرة على الراوتر", "لم يُرصد أي منفذ عالي الخطورة.", Severity.OK)

        out += Finding(
            "المنافذ المفتوحة على الراوتر ($gw)",
            if (open.isEmpty()) "لا شيء" else open.joinToString("، ") { "$it/${PORTS[it]}" },
            Severity.INFO
        )
        return out
    }

    // ---------- الأجهزة ----------

    private suspend fun devicesFinding(ctx: Context): Finding {
        val ip = ownIp(ctx) ?: return Finding("الأجهزة المتصلة", "تعذّر تحديد عنوان الجهاز.", Severity.INFO)
        val prefix = ip.substringBeforeLast('.')
        val sem = Semaphore(40)
        val alive = coroutineScope {
            (1..254).map { i ->
                async(Dispatchers.IO) {
                    sem.withPermit {
                        val h = "$prefix.$i"
                        if (try { InetAddress.getByName(h).isReachable(300) } catch (e: Exception) { false }) h else null
                    }
                }
            }.awaitAll().filterNotNull()
        }
        return Finding(
            "الأجهزة المتصلة: ${alive.size}",
            alive.joinToString("، ") + "\nراجع القائمة وتأكد أنك تعرف كل جهاز فيها.",
            Severity.INFO
        )
    }

    // ---------- أدوات مساعدة ----------

    private suspend fun isOpen(host: String, port: Int, timeoutMs: Int = 400): Boolean =
        withContext(Dispatchers.IO) {
            try {
                Socket().use { it.connect(InetSocketAddress(host, port), timeoutMs); true }
            } catch (e: Exception) {
                false
            }
        }

    private fun intToIp(i: Int) =
        "${i and 0xff}.${(i shr 8) and 0xff}.${(i shr 16) and 0xff}.${(i shr 24) and 0xff}"

    private fun gatewayIp(ctx: Context): String? {
        val wm = ctx.getSystemService(Context.WIFI_SERVICE) as WifiManager
        val g = wm.dhcpInfo?.gateway ?: 0
        return if (g == 0) null else intToIp(g)
    }

    private fun ownIp(ctx: Context): String? {
        val wm = ctx.getSystemService(Context.WIFI_SERVICE) as WifiManager
        val a = wm.dhcpInfo?.ipAddress ?: 0
        return if (a == 0) null else intToIp(a)
    }
}
