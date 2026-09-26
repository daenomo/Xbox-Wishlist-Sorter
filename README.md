# Xbox Wishlist Discount Sorter

Tampermonkeyなどのユーザースクリプト環境で動作し、Xboxの欲しい物リスト（ウィッシュリスト）の価格情報を解析して**割引率の自動表示**および**ソート機能**を追加するスクリプトです。

## 主な機能

* **割引率の自動計算・表示**: セール対象のゲームについて、元値とセール価格から割引率を自動計算し、価格の横に `(XX% OFF)` と表示します。
* **割引率順ソート**: リスト上部の「[割引率順]」リンクをクリックすることで、割引率が高い（お得な）順にアイテムを並び替えます。
* **セール価格安い順ソート**: 「[セール価格安い順]」リンクをクリックすることで、実際のセール価格が安い順に並び替えます。

## 動作環境

* [Tampermonkey](https://www.tampermonkey.net/?utm_source=gemini) などのユーザースクリプトマネージャーがインストールされているブラウザ
* 対象URL: `[https://www.xbox.com/*/wishlist](https://www.xbox.com/*/wishlist)*`（Xboxの欲しい物リストページ）

## インストール方法

1. ブラウザに Tampermonkey などの拡張機能をインストールします。
2. 拡張機能のメニューから「新規スクリプトの追加」を選択します。
3. 下記のコードをすべて貼り付けて保存（`Ctrl + S` / `Cmd + S`）します。
4. Xboxの欲しい物リストページ（`[https://www.xbox.com/ja-JP/wishlist](https://www.xbox.com/ja-JP/wishlist)` 等）を開くと、自動的に機能が有効になります。

## ユーザースクリプトコード

```javascript
// ==UserScript==
// @name         Xbox Wishlist Sorter
// @namespace    http://tampermonkey.net/
// @version      1.2
// @description  Xboxの欲しい物リストに割引率を表示し、割引率順またはセール価格の安い順にソートします
// @match        https://www.xbox.com/*/wishlist*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    function processItems() {
        let parent = document.evaluate('//*[@id="PageContent"]/div/div/div[2]', document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue;
        if (!parent) return;

        let items = Array.from(parent.children);
        items.forEach(el => {
            if (el.dataset.discountProcessed) return;

            let spans = Array.from(el.querySelectorAll('span')).filter(s => s.textContent.includes('¥'));
            let prices = [];
            spans.forEach(s => {
                let t = s.textContent.trim();
                let m = t.replace(/[^\d]/g, '');
                if (m) prices.push({ el: s, val: Number(m) });
            });

            if (prices.length >= 2) {
                prices.sort((a, b) => b.val - a.val);
                let original = prices[0].val;
                let discounted = prices[1].val;

                if (original > discounted && original > 0) {
                    let discountRate = Math.round((1 - discounted / original) * 100);
                    let container = prices[1].el.parentNode;
                    
                    if (!container.querySelector('.custom-discount-rate') && !container.textContent.includes('OFF')) {
                        let badge = document.createElement('span');
                        badge.className = 'custom-discount-rate';
                        badge.style.cssText = 'margin-left:8px;color:#ff4081;font-weight:bold;';
                        badge.textContent = `(${discountRate}% OFF)`;
                        container.appendChild(badge);
                    }
                    el.dataset.discountRate = discountRate;
                    el.dataset.discountedPrice = discounted;
                } else {
                    el.dataset.discountRate = -1;
                    el.dataset.discountedPrice = 999999;
                }
            } else {
                el.dataset.discountRate = -1;
                let singlePrice = prices.length === 1 ? prices[0].val : 999999;
                el.dataset.discountedPrice = singlePrice;
            }
            el.dataset.discountProcessed = "true";
        });
    }

    function sortItems(type) {
        processItems();
        let parent = document.evaluate('//*[@id="PageContent"]/div/div/div[2]', document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue;
        if (!parent) return;

        let items = Array.from(parent.children);
        let validItems = [], otherItems = [];

        items.forEach(el => {
            let rate = Number(el.dataset.discountRate || -1);
            if (rate >= 0) {
                validItems.push(el);
            } else {
                otherItems.push(el);
            }
        });

        if (type === 'rate') {
            validItems.sort((a, b) => Number(b.dataset.discountRate) - Number(a.dataset.discountRate));
        } else if (type === 'price') {
            validItems.sort((a, b) => Number(a.dataset.discountedPrice) - Number(b.dataset.discountedPrice));
        }

        otherItems.sort((a, b) => a.textContent.trim().localeCompare(b.textContent.trim()));

        while (parent.firstChild) parent.removeChild(parent.firstChild);
        validItems.forEach(el => parent.appendChild(el));
        otherItems.forEach(el => parent.appendChild(el));
    }

    function addSortLinks() {
        if (document.getElementById('sort-container')) return;

        let allElements = document.querySelectorAll('*');
        let targetEl = null;
        for (let el of allElements) {
            if (el.childNodes.length === 1 && el.childNodes[0].nodeType === Node.TEXT_NODE && el.textContent.trim() === 'XBOX 欲しい物リスト') {
                targetEl = el;
                break;
            }
        }

        if (targetEl) {
            let container = document.createElement('span');
            container.id = 'sort-container';
            container.style.cssText = 'margin-left: 15px; font-size: 0.7em; vertical-align: middle; font-weight: normal;';

            let linkRate = document.createElement('a');
            linkRate.href = '#';
            linkRate.textContent = '[割引率順]';
            linkRate.style.cssText = 'color: #0078d4; text-decoration: underline; cursor: pointer; margin-right: 10px;';
            linkRate.addEventListener('click', (e) => {
                e.preventDefault();
                sortItems('rate');
            });

            let linkPrice = document.createElement('a');
            linkPrice.href = '#';
            linkPrice.textContent = '[セール価格安い順]';
            linkPrice.style.cssText = 'color: #0078d4; text-decoration: underline; cursor: pointer;';
            linkPrice.addEventListener('click', (e) => {
                e.preventDefault();
                sortItems('price');
            });

            container.appendChild(linkRate);
            container.appendChild(linkPrice);
            targetEl.appendChild(container);
        }
    }

    const observer = new MutationObserver(() => {
        processItems();
        addSortLinks();
    });

    observer.observe(document.body, { childList: true, subtree: true });

    window.addEventListener('load', () => {
        setTimeout(() => {
            processItems();
            addSortLinks();
        }, 1000);
    });
})();

```

## ライセンス

MIT License
