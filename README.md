// ==UserScript==
// @name         Crypto Transactions Telegram Notifier
// @namespace    http://tampermonkey.net/
// @version      4.1
// @description  Traccia nuove transazioni su BullX e inviale a Telegram
// @match        https://neo.bullx.io/wallet-tracker*   // Lo script si attiva su questa pagina specifica
// @grant        GM_xmlhttpRequest                      // Permette di fare richieste HTTP esterne (serve per Telegram)
// @run-at       document-end                           // Lo script parte solo quando la pagina è completamente caricata
// ==/UserScript==

(function () {
    'use strict';

    console.log("✅ Script avviato correttamente!"); // Messaggio visibile nella console quando lo script parte

    // === CONFIG TELEGRAM ===
    const TELEGRAM = {
        chatId: "OSCURATO PER PRIVACY",                           // ID della chat Telegram dove inviare i messaggi
        API: "OSCURATO PER PRIVACY", // Token del bot Telegram
        apiUrl: "https://api.telegram.org/bot"           // URL base per le API Telegram
    };

    // Set che tiene traccia delle transazioni già inviate, per evitare duplicati
    const processedTx = new Set();

    // Funzione per estrarre i dati utili da una riga della tabella delle transazioni
    function extractRowData(row) {
        const type = row.querySelector('span[style*="background-color"]')?.innerText.trim() || 'N/A'; // Tipo operazione (Buy/Sell)
        return {
            age: row.querySelector('.b-table-cell:nth-child(1) .text-green-700')?.innerText.trim() || 'N/A', // Età transazione
            token: row.querySelector('.b-table-cell:nth-child(2) .uppercase')?.innerText.trim() || 'N/A',     // Nome token
            type: type,                                                                                       // Tipo operazione (B o altro)
            amount: row.querySelector('.b-table-cell:nth-child(3) .text-green-700')?.innerText.trim() || 'N/A', // Quantità acquistata/venduta
            mcap: row.querySelector('.b-table-cell:nth-child(4) .text-green-700')?.innerText.trim() || 'N/A',  // Market cap
            maker: row.querySelector('.text-yellow-500')?.innerText.trim() || 'N/A',                          // Nome del trader
            tokenAddress: row.querySelector('.b-table-cell:nth-child(2) a')?.href.split('address=')[1] || 'N/A' // Estrazione dell'indirizzo token
        };
    }

    // Funzione che costruisce il messaggio e lo invia su Telegram
    function sendToTelegram(data) {
        const action = data.type === 'B' ? 'acquistato' : 'venduto'; // Determina il tipo di azione
        const message = `🚀 Il trader *${data.maker}* ha appena *${action}* il token *${data.token}* \n\n🔗 [Vai al token](https://dexscreener.com/search?q=${data.tokenAddress})`;

        // Chiamata HTTP POST verso Telegram API
        GM_xmlhttpRequest({
            method: "POST",
            url: `${TELEGRAM.apiUrl}${TELEGRAM.botToken}/sendMessage`, // Costruzione URL API
            data: JSON.stringify({
                chat_id: TELEGRAM.chatId,
                text: message,
                parse_mode: "Markdown"  // Telegram formatta il testo con Markdown
            }),
            headers: {
                "Content-Type": "application/json" // Tipo di contenuto inviato
            },
            onload: function (response) {
                console.log("📤 Messaggio Telegram inviato:", response.responseText); // Log in console del risultato
            }
        });
    }

    // Funzione che controlla tutte le righe attualmente presenti e invia quelle nuove
    function checkNewTransactions() {
        const rows = document.querySelectorAll('.b-table-row'); // Seleziona tutte le righe delle transazioni

        rows.forEach(row => {
            const data = extractRowData(row); // Estrae i dati
            const txId = `${data.token}_${data.amount}_${data.maker}`; // Identificativo unico della transazione

            // Se non è già stata processata, la inviamo a Telegram
            if (!processedTx.has(txId)) {
                processedTx.add(txId); // Segna come già inviata
                sendToTelegram(data);  // Invio effettivo
            }
        });
    }

    // Controlla le nuove transazioni ogni 3 secondi (polling)
    setInterval(checkNewTransactions, 3000);

    // Usa un MutationObserver per rilevare aggiornamenti dinamici alla pagina
    const observer = new MutationObserver(checkNewTransactions);
    observer.observe(document.body, { childList: true, subtree: true }); // Osserva tutta la pagina per cambiamenti
})();
