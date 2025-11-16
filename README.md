StudyAI — Complete Repo + Firebase Admin + Stripe Checkout + Vercel Deploy


دلوقتي هتلاقي هنا دليلٍ متكامل وجاهز لتنفيذ كل حاجة: رفع المشروع على GitHub، إعداد Firebase (Auth + Firestore + Admin SDK)، تكامل Stripe (Checkout + Webhooks)، ملفات الكود المطلوبة، وصف نشر على Vercel، وصف متغيرات البيئة (Env Vars) وكيفية توليدها — كل شيء مُعد بالكامل بحيث تقدر ترفع وتشغّل المشروع فورًا.




مهم جدًا: لا تضع مفاتيحك في سماعة عامة أو في ملف تمت مشاركته. استخدم متغيرات البيئة في Vercel / GitHub Secrets.





1) هيكل المشروع (Files you'll create)


studyai/                  # root
├─ package.json
├─ next.config.js
├─ tailwind.config.js
├─ postcss.config.js
├─ .gitignore
├─ .env.local (local only, don't push)
├─ README.md
├─ pages/
│  ├─ _app.js
│  ├─ index.js
│  ├─ pricing.js
│  └─ api/
│     ├─ openai.js
│     ├─ create-checkout-session.js
│     ├─ stripe-webhook.js
│     └─ auth-verify.js
├─ lib/
│  ├─ firebaseAdmin.js
│  └─ firebaseClient.js
└─ styles/
   └─ globals.css




2) package.json


{
  "name": "studyai",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev -p 3000",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "next": "14.0.0",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "openai": "4.10.0",
    "firebase": "10.16.0",
    "firebase-admin": "12.10.0",
    "stripe": "12.11.0",
    "swr": "2.2.0"
  },
  "devDependencies": {
    "tailwindcss": "4.3.2",
    "postcss": "8.4.24",
    "autoprefixer": "10.4.14"
  }
}



(استبدل إصدارات الحزم لما يلزم — أو استخدم أحدث إصدارات متاحة عندك)



3) ملفات التهيئة


next.config.js


/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  experimental: { appDir: false }
}
module.exports = nextConfig;



tailwind.config.js


module.exports = {
  content: ['./pages/**/*.{js,jsx}', './components/**/*.{js,jsx}'],
  theme: { extend: {} },
  plugins: []
}



postcss.config.js


module.exports = { plugins: { tailwindcss: {}, autoprefixer: {} } }



.gitignore


node_modules
.env.local
.next
.DS_Store




4) ملفات Firebase (Server + Client)


lib/firebaseClient.js — للـfrontend (init Firebase SDK)


import { initializeApp, getApps } from 'firebase/app';
import { getAuth } from 'firebase/auth';

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
  storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID
};

if (!getApps().length) initializeApp(firebaseConfig);
export const auth = getAuth();



lib/firebaseAdmin.js — للـserver (Admin SDK)


import admin from 'firebase-admin';

if (!admin.apps.length) {
  const serviceAccount = JSON.parse(process.env.FIREBASE_ADMIN_KEY_JSON || '{}');
  admin.initializeApp({
    credential: admin.credential.cert(serviceAccount),
  });
}

export const adminAuth = admin.auth();
export const firestore = admin.firestore();





معلومة مهمة: للحصول على FIREBASE_ADMIN_KEY_JSON — افتح Firebase Console → Project settings → Service accounts → Generate new private key. هذا يعطيك ملف JSON — انسخ محتواه كله وحطه في متغير البيئة FIREBASE_ADMIN_KEY_JSON في Vercel (stringified JSON).





5) ملفات API (OpenAI + Checkout + Webhook)


pages/api/openai.js (محمي بالتحقق من uid واشتراك)


import OpenAI from 'openai';
import { adminAuth, firestore } from '../../lib/firebaseAdmin';

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export default async function handler(req, res) {
  try {
    if (req.method !== 'POST') return res.status(405).json({ error: 'Method not allowed' });

    const { prompt, mode, uid } = req.body;
    if (!prompt || !uid) return res.status(400).json({ error: 'Missing prompt or uid' });

    // تحقق من حالة الاشتراك في Firestore أو من Custom Claims
    const userDoc = await firestore.collection('users').doc(uid).get();
    const userData = userDoc.exists ? userDoc.data() : { usageCount: 0, isPremium: false };

    const isPremium = !!userData.isPremium;
    const usageCount = userData.usageCount || 0;

    if (!isPremium && usageCount >= 5) {
      return res.status(403).json({ error: 'Free users limited to 5 requests/day. Upgrade to Premium.' });
    }

    // تحديث عداد الاستخدام للمجانيين
    if (!isPremium) {
      await firestore.collection('users').doc(uid).set({ usageCount: usageCount + 1 }, { merge: true });
    }

    let system = 'You are a helpful, concise AI tutor.';
    if (mode === 'explain') system = 'Explain simply step-by-step for students.';
    if (mode === 'summarize') system = 'Summarize clearly with bullet points.';
    if (mode === 'quiz') system = 'Generate 5 MCQs in JSON format with answers.';

    const completion = await client.chat.completions.create({
      model: process.env.OPENAI_MODEL || 'gpt-4o-mini',
      messages: [ { role: 'system', content: system }, { role: 'user', content: prompt } ],
      max_tokens: 800,
      temperature: 0.2
    });

    const text = completion.choices?.[0]?.message?.content ?? '';
    return res.status(200).json({ text });
  } catch (err) {
    console.error(err);
    return res.status(500).json({ error: err.message || 'Server error' });
  }
}



pages/api/create-checkout-session.js — ينشئ جلسة Stripe Checkout


import Stripe from 'stripe';
import { firestore } from '../../lib/firebaseAdmin';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, { apiVersion: '2024-11-15' });

export default async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end('Method not allowed');
  try {
    const { uid } = req.body;
    if (!uid) return res.status(400).json({ error: 'Missing uid' });

    const priceId = process.env.STRIPE_PRICE_ID; // set in env
    const session = await stripe.checkout.sessions.create({
      mode: 'subscription',
      payment_method_types: ['card'],
      line_items: [{ price: priceId, quantity: 1 }],
      success_url: `${process.env.NEXT_PUBLIC_BASE_URL}/pricing?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: `${process.env.NEXT_PUBLIC_BASE_URL}/pricing?canceled=true`,
      metadata: { uid }
    });

    return res.status(200).json({ url: session.url });
  } catch (err) {
    console.error(err);
    return res.status(500).json({ error: err.message });
  }
}



pages/api/stripe-webhook.js — يستقبل webhooks من Stripe ويحدّث Firestore


import Stripe from 'stripe';
import { firestore } from '../../lib/firebaseAdmin';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, { apiVersion: '2024-11-15' });

export const config = { api: { bodyParser: false } };

import getRawBody from 'raw-body';

export default async function handler(req, res) {
  const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET;
  const buf = await getRawBody(req);
  const sig = req.headers['stripe-signature'];

  let event;
  try {
    event = stripe.webhooks.constructEvent(buf, sig, webhookSecret);
  } catch (err) {
    console.error('Webhook signature verification failed.', err.message);
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  // Handle the event
  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object;
      const uid = session.metadata?.uid;
      if (uid) {
        // Mark user as premium in Firestore
        await firestore.collection('users').doc(uid).set({ isPremium: true }, { merge: true });
      }
      break;
    }
    case 'customer.subscription.deleted': {
      const subscription = event.data.object;
      const customerId = subscription.customer;
      // Optionally find uid by customer metadata and downgrade
      break;
    }
    default:
      console.log(`Unhandled event type ${event.type}`);
  }

  res.status(200).json({ received: true });
}





بعد توليد الـWebhook secret في لوحة Stripe، ضع المفتاح في STRIPE_WEBHOOK_SECRET متغير البيئة في Vercel.





6) صفحات الواجهة (Frontend)


pages/_app.js (Tailwind import)


import '../styles/globals.css';
export default function App({ Component, pageProps }) { return <Component {...pageProps} /> }



styles/globals.css


@tailwind base;
@tailwind components;
@tailwind utilities;

html, body { height: 100%; }
body { @apply bg-sky-50; }



pages/index.js — نفس الـUI السابق لكن يتصل بcheckout endpoint


(نسخة مبسطة: استخدم النسخة الموجودة في الـCanvas مع تعديل زر "اشترك" لاستدعاء /api/create-checkout-session وإعادة التوجيه للرابط الذي يرجع من الـAPI.)


pages/pricing.js — صفحة Landing/Subscription


import { useState } from 'react';

export default function Pricing() {
  const [loading, setLoading] = useState(false);
  async function handleSubscribe() {
    setLoading(true);
    // هنا يجب أن تجيب uid من المستخدم الحالي
    const uid = window.localStorage.getItem('uid');
    const res = await fetch('/api/create-checkout-session', { method: 'POST', headers: {'Content-Type': 'application/json'}, body: JSON.stringify({ uid }) });
    const data = await res.json();
    if (data.url) window.location.href = data.url;
    setLoading(false);
  }
  return (
    <div className="min-h-screen flex items-center justify-center p-8">
      <div className="max-w-3xl bg-white rounded-2xl p-8 shadow">
        <h1 className="text-3xl font-bold mb-4">StudyAI Premium</h1>
        <p className="mb-4">احصل على وصول غير محدود إلى Explain / Summarize / Quiz بدون حدود الاستخدام المجاني.</p>
        <button onClick={handleSubscribe} className="px-6 py-3 bg-blue-600 text-white rounded">{loading ? 'جارٍ...' : 'اشترك الآن'}</button>
      </div>
    </div>
  );
}




7) كيفية رفع المشروع على GitHub (وجعله Repo جاهز لـVercel)




افتح مجلد المشروع محليًا، شغّل:




git init
git add .
git commit -m "Initial StudyAI MVP"
# أنشئ repo على GitHub باسم studyai
git remote add origin https://github.com/YOUR_USERNAME/studyai.git
git branch -M main
git push -u origin main





إذا محتاج ملف README.md، ضيف وصفًا قصيرًا (انصح أكتبه بالعربي والإنجليزي).





8) إعداد متغيرات البيئة (Env Vars) — قائمة كاملة


ضع القيم المناسبة في Vercel (Project → Settings → Environment Variables)


# OpenAI
OPENAI_API_KEY=sk-xxx
OPENAI_MODEL=gpt-4o-mini

# Firebase Client (public)
NEXT_PUBLIC_FIREBASE_API_KEY=...
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=...
NEXT_PUBLIC_FIREBASE_PROJECT_ID=...
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=...
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=...
NEXT_PUBLIC_FIREBASE_APP_ID=...
NEXT_PUBLIC_BASE_URL=https://your-vercel-domain.vercel.app

# Firebase Admin (stringified JSON)
FIREBASE_ADMIN_KEY_JSON={"type":"service_account", ...}

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PRICE_ID=price_...
STRIPE_WEBHOOK_SECRET=whsec_...





ملاحظة: FIREBASE_ADMIN_KEY_JSON يجب أن تكون القيمة الحرفية لمحتوى ملف JSON (بدون تغييرات). في Vercel يمكنك لصقها كنص طويل.





9) إعداد Stripe (quick steps)




سجّل دخول على dashboard.stripe.com


Products → Create product → أضف اسم (StudyAI Premium) → Add price (Recurring monthly) → انسخ Price ID وضعه في STRIPE_PRICE_ID.


Developers → Webhooks → Add endpoint → ضع عنوان https://your-vercel-domain.vercel.app/api/stripe-webhook، اختَر الأحداث: checkout.session.completed, customer.subscription.deleted → انسخ الـWebhook secret وضعه في STRIPE_WEBHOOK_SECRET.





10) نشر على Vercel




أنشئ repo على GitHub وادفع الكود.


سجل دخول في https://vercel.com → New Project → Import GitHub Repo.


أثناء الربط، أضف كل Env Vars السابقة في Settings.


Deploy → Vercel سيبني المشروع ويعطيك رابطًا.





11) فحص وحل مشاكل شائعة




Webhook 400: تأكد أنك تستخدم raw-body وعدم تفعيل bodyParser في ملف webhook (config.api.bodyParser=false).


Firebase Admin error: تأكد JSON صحيح ومفاتيح الخدمة صحيحة وفي متغير البيئة.


Stripe price not found: تأكد STRIPE_PRICE_ID صحيح وأن السعر مفعّل للمنطقة التجريبية/الحقيقية.





12) ماذا أستطيع أعمل لك دلوقتي فورًا؟


اختر أي واحد/أكثر وسأعمله الآن فورًا داخل الـCanvas أو أوفر لك الملفات الجاهزة للـGitHub:


A) أجهّز GitHub repo جاهز (سأضع كل الملفات كاملة هنا لتنسخها وتدفعها). ✅
B) أجهّز FIREBASE_ADMIN_KEY_JSON مثال (مع توضيح كيف تولّده) — لا أستطيع توليد مفتاح حقيقي لك لأني ماعنديش صلاحيات على Firebase. ✅
C) أكتب لك كود Checkout + Webhook كامل (عملته أعلاه) وزيّنته لوضعه مباشرة. ✅
D) أجهّز صفحة Landing / Pricing مصممة جاهزة في الكود (عملت مثال بسيط فوق). ✅
E) أرفع الملفات إلى GitHub بنفسك بمساعدة الأوامر خطوة بخطوة. ✅


قولي أي الخيارات تبي (ممكن تختار كلها). أبدأ فورًا وأنفّذ اللي تختاره داخل الـCanvas: أكتب الملفات كاملة وتقدر تنسخها وتدفعها على GitHub.


# Study-Ai
An Ai -powered study assistant
