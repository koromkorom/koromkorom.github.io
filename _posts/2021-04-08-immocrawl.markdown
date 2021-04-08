---
layout: post
title:  "Housing Web Crawler"
date:   2021-04-08 14:00:00 +0100
categories: javascript
---

I had the idea of having my own web crawler for Immobilienscout24.de. Whenever there are new postings, the crawler pushes its data into firestore. 
When there are changes, the data is changed inside firestore db. 

Lets checkout the google cloud function script: 

```javascript
// Partly from here: https://cloud.google.com/blog/products/gcp/introducing-headless-chrome-support-in-cloud-functions-and-app-engine

const functions = require('firebase-functions');
const admin = require('firebase-admin');
const puppeteer = require('puppeteer')

admin.initializeApp(functions.config().firebase);
let db = admin.firestore();
let immos;
let page;

```

These are just dependencies. puppeteer is the web crawler library of my choice. I don't have much to say about it, besides that 
it works well with google cloud functions. Also the firesore db interface is defined here. 

```javascript

async function getBrowserPage() {
    // Launch headless Chrome. Turn off sandbox so Chrome can run under root.
    const browser = await puppeteer.launch({ args: ['--no-sandbox'] });
    return browser.newPage();
}

```

This is the first function. This launches the web page in a headless browser,

```javascript

exports.screenshot = functions.https.onRequest(async (req, res) => {
    const url = req.query.url;

    if (!url) {
        return res.send('Please provide URL as GET parameter, for example: <a href="?url=https://example.com">?url=https://example.com</a>');
    }

    if (!page) {
        page = await getBrowserPage();
    }

    await page.goto(url);
    const imageBuffer = await page.screenshot();
    res.set('Content-Type', 'image/png');
    res.send(imageBuffer);
    return null;
});

```
The second function is just for testing. Is screenshot says a lot about whats going on in a headless browser. Believe me. 

```javascript

exports.getImmo = functions.https.onRequest(async (req, res) => {
    let allAptms = await db.collection('aptms').get()
        .catch(err => {
            console.log('Error getting documents', err);
        });
    if (!allAptms) {
        res.send("Couldn't manage to get appartments from db")
    } else {
        res.send(allAptms.docs.map(doc => doc.data()))
    }
});

```

This returns the whole database as yaml. 

```javascript

/* eslint no-await-in-loop: "off" */
exports.callImmo = functions.runWith({ timeoutSeconds: 300, memory: '1GB' }).pubsub.schedule('0 7-22 * * *').timeZone('Europe/Berlin').onRun(async (context) => {
    for (i = 1; i < 5; i++) {
        let url = 'https://www.immobilienscout24.de/Suche/S-2/P-' + i + '/Wohnung-Kauf/Berlin/Berlin/-/3,00-/70,00-/EURO--600000,00/-/-/-/-/-/true';
        console.log(url);
        if (!page) {
            page = await getBrowserPage();
        }
        await page.goto(url, {
            waitUntil: 'domcontentloaded',
            timeout: 60000
        });
        await page.waitForSelector('#listings')
        immos = await page.$$eval('.result-list__listing', anchors => {
            return anchors.filter(anchor => anchor.getAttribute('data-id') !== null).map(anchor =>
                [
                    anchor.getAttribute('data-id'),
                    anchor.getElementsByTagName('h5')[0].textContent,
                    anchor.getElementsByClassName('result-list-entry__address')[0].textContent,
                    anchor.getElementsByTagName('dd')[0].textContent,
                    anchor.getElementsByTagName('dd')[1].textContent,
                    anchor.getElementsByTagName('dd')[2].textContent,
                ])//.slice(0, 10)
        })
        console.log(immos);
        immos.forEach(element => {
            let docRef = db.collection('aptms').doc(element[0]);
            docRef.get().then(function (doc) {
                if (doc.exists) {
                    if (doc.get('price') !== element[3]) {
                        let addDoc = doc.ref.update({
                            price: element[3],
                            newprice: true,
                        });
                    }
                    return null;
                } else {
                    let addDoc = doc.ref.set({
                        id: element[0],
                        name: element[1],
                        loc: element[2],
                        price: element[3],
                        size: element[4],
                        rooms: element[5],
                        fav: false,
                        newprice: false,
                        site: "https://www.immobilienscout24.de/",
                    });
                    return null;
                }
            }).catch(function (error) {
                console.log("Error getting document:", error);
            });
        });
    }
    return null;
});
```

This last function is the heart of the programm. It runs regularly and checks the page for new postings. This needed a lot of tweaking. Especially with the timeouts. Also immoscout checks after some time with a I'm no robot captcha, which kind of breaks the crawler, but this doesn't happen a lot.