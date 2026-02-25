// 1. DATABASE SCHEMA (Run this in your Supabase SQL Editor)
/*
create table quotes (
  id uuid default uuid_generate_v4() primary key,
  text text not null,
  author text,
  explanation text,
  action_step text,
  created_at timestamp with time zone default now()
);
*/

// 2. THE MAIN PAGE (src/app/page.tsx)
import React from 'react';
import { QuoteCard } from '@/components/QuoteCard';
import { supabase } from '@/lib/supabase';

export default async function MomentumPage() {
  // Fetch the latest quote from the last 24 hours
  const { data: quote } = await supabase
    .from('quotes')
    .select('*')
    .order('created_at', { ascending: false })
    .limit(1)
    .single();

  return (
    <main className="min-h-screen flex items-center justify-center bg-[radial-gradient(ellipse_at_top,_var(--tw-gradient-stops))] from-gray-900 via-black to-black text-white p-6">
      {/* Background Ambient Glow */}
      <div className="absolute top-0 left-1/2 -translate-x-1/2 w-full h-full max-w-4xl opacity-30 blur-[120px] bg-blue-600 rounded-full -z-10" />
      
      <div className="w-full max-w-2xl">
        <nav className="flex justify-between items-center mb-12">
          <h1 className="text-xl font-bold tracking-tighter italic">MOMENTUM AI</h1>
          <div className="h-2 w-2 rounded-full bg-green-500 animate-pulse" />
        </nav>

        {quote ? (
          <QuoteCard quote={quote} />
        ) : (
          <div className="text-center opacity-50">Initializing Momentum...</div>
        )}

        <footer className="mt-12 text-center text-xs text-white/40 tracking-widest uppercase">
          New Wisdom in 24h • Powered by AI
        </footer>
      </div>
    </main>
  );
}

// 3. THE UI COMPONENT (src/components/QuoteCard.tsx)
'use client';
import { motion } from 'framer-motion';
import { Share2, Bookmark } from 'lucide-react';

export function QuoteCard({ quote }: { quote: any }) {
  return (
    <motion.div 
      initial={{ opacity: 0, scale: 0.95 }}
      animate={{ opacity: 1, scale: 1 }}
      className="relative group p-1 rounded-[32px] bg-gradient-to-b from-white/20 to-transparent"
    >
      <div className="bg-black/60 backdrop-blur-3xl p-10 rounded-[30px] border border-white/10">
        <span className="text-blue-400 text-xs font-bold tracking-[0.3em] uppercase">Daily Momentum</span>
        
        <h2 className="mt-6 text-3xl md:text-5xl font-light leading-tight tracking-tight">
          "{quote.text}"
        </h2>
        
        <p className="mt-4 text-white/50 font-medium">— {quote.author || 'Unknown'}</p>

        <div className="mt-10 pt-8 border-t border-white/5 space-y-6">
          <div>
            <h4 className="text-[10px] uppercase tracking-widest text-white/30 mb-2">The Insight</h4>
            <p className="text-lg text-white/80 leading-relaxed">{quote.explanation}</p>
          </div>

          <div className="bg-white/5 p-5 rounded-2xl border border-white/10">
            <h4 className="text-[10px] uppercase tracking-widest text-blue-400 mb-2 font-bold">Today's Micro-Step</h4>
            <p className="text-white/90">{quote.action_step}</p>
          </div>
        </div>

        <div className="mt-8 flex gap-4">
          <button className="flex-1 bg-white text-black py-3 rounded-xl font-bold hover:bg-blue-400 transition-colors flex items-center justify-center gap-2">
            <Bookmark size={18} /> Save
          </button>
          <button className="p-3 rounded-xl bg-white/5 border border-white/10 hover:bg-white/10 transition-all">
            <Share2 size={20} />
          </button>
        </div>
      </div>
    </motion.div>
  );
}

// 4. THE API ROUTE (src/app/api/cron/route.ts)
import { OpenAI } from 'openai';
import { createClient } from '@supabase/supabase-js';

export async function GET(request: Request) {
  const supabase = createClient(process.env.SUPABASE_URL!, process.env.SUPABASE_SERVICE_ROLE_KEY!);
  const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

  // Fetch unique quote
  const res = await fetch('https://api.quotable.io/random');
  const data = await res.json();

  // AI Augmentation
  const aiResponse = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{
      role: "system",
      content: "You are a high-performance coach. Provide a 2-line explanation and 1 clear action step for this quote. Format: Explanation | Action"
    }, { role: "user", content: data.content }]
  });

  const [exp, step] = aiResponse.choices[0].message.content!.split('|');

  // Store in DB
  const { error } = await supabase.from('quotes').insert({
    text: data.content,
    author: data.author,
    explanation: exp.trim(),
    action_step: step.trim()
  });

  return Response.json({ success: !error });
}
