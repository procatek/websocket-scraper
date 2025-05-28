// ==UserScript==
// @name         Crypto Transactions Telegram Notifier
// @namespace    http://tampermonkey.net/
// @version      4.1
// @description  Traccia nuove transazioni su BullX e inviale a Telegram
// @match        https://neo.bullx.io/wallet-tracker*
// @grant        GM_xmlhttpRequest
// @run-at       document-end
// ==/UserScript==

(function () {
    'use strict';

    console.log("✅ Script avviato correttamente!");

    // Configurazione Telegram
    const TELEGRAM = {
        chatId: "-4806100750",
        botToken: "7461428790:AAFFJFOKrqICB7IPPv362czVCwTRTNrNvcc",
        apiUrl: "https://api.telegram.org/bot"
    };

    const processedTx = new Set();

    function extractRowData(row) {
        const type = row.querySelector('span[style*="background-color"]')?.innerText.trim() || 'N/A';
        return {
            age: row.querySelector('.b-table-cell:nth-child(1) .text-green-700')?.innerText.trim() || 'N/A',
            token: row.querySelector('.b-table-cell:nth-child(2) .uppercase')?.innerText.trim() || 'N/A',
            type: type,
            amount: row.querySelector('.b-table-cell:nth-child(3) .text-green-700')?.innerText.trim() || 'N/A',
            mcap: row.querySelector('.b-table-cell:nth-child(4) .text-green-700')?.innerText.trim() || 'N/A',
            maker: row.querySelector('.text-yellow-500')?.innerText.trim() || 'N/A',
            tokenAddress: row.querySelector('.b-table-cell:nth-child(2) a')?.href.split('address=')[1] || 'N/A'
        };
    }

    function sendToTelegram(data) {
        const action = data.type === 'B' ? 'acquistato' : 'venduto';
        const message = `🚀 Il trader *${data.maker}* ha appena *${action}* il token *${data.token}* \n\n🔗 [Vai al token](https://dexscreener.com/search?q=${data.tokenAddress})`;

        GM_xmlhttpRequest({
            method: "POST",
            url: `${TELEGRAM.apiUrl}${TELEGRAM.botToken}/sendMessage`,
            data: JSON.stringify({
                chat_id: TELEGRAM.chatId,
                text: message,
                parse_mode: "Markdown"
            }),
            headers: {
                "Content-Type": "application/json"
            },
            onload: function (response) {
                console.log("📤 Messaggio Telegram inviato:", response.responseText);
            }
        });
    }

    function checkNewTransactions() {
        const rows = document.querySelectorAll('.b-table-row');
        rows.forEach(row => {
            const data = extractRowData(row);
            const txId = `${data.token}_${data.amount}_${data.maker}`;

            if (!processedTx.has(txId)) {
                processedTx.add(txId);
                sendToTelegram(data);
            }
        });
    }

    // Esegui polling ogni 3 secondi
    setInterval(checkNewTransactions, 3000);

    // Osservatore per rilevare cambiamenti dinamici
    const observer = new MutationObserver(checkNewTransactions);
    observer.observe(document.body, { childList: true, subtree: true });
})();
