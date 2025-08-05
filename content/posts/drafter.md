---
title: Drafter
description: "Use AI to auto-draft positive and negative emails replies, then you can choose your own adventure."
date: 2025-08-05
draft: true
authors:
  - name: Andy Casey
    link: https://astrowizici.st
    image: ../../me.png
tags:
  - google-apps-script
  - productivity
  - automation
  - email
---

Who asked for emails to be invented? I certainly didn't.

Sometimes I feel I would be a lot faster to reply to emails if I just had a little stem drafted for me that I could edit and send. Let's automate that with Google Apps Script.

This script will monitor your Gmail inbox every few minutes. When there is a new email, it will check if it is specifically addressed to me and if it looks like it needs my response. If so, it will use Google's Gemini AI model to generate two draft responses:
1. A positive response that concisely addresses what I am being asked, without setting strict timescales.
2. An extremely polite "no" response.

The drafts will be created in Gmail, ready for me to review and send. This way, I can quickly choose which response to send without having to start from scratch.

And so my inbox doesn't get too cluttered with old drafts, the script will also delete any drafts older than about a month.

Here's the code:

```javascript
/**
 * Email Auto-Draft Generator with Gemini AI
 * This script monitors emails and generates draft responses using Google's Gemini AI
 */

const GEMINI_API_KEY = '...'; // Get from Google AI Studio
const GEMINI_API_URL = 'https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent';

/**
 * Main function that runs every minute via trigger
 */
function processNewEmails() {
  try {
    console.log('Starting email processing...');
    
    // Get the last processed timestamp
    const lastProcessed = getLastProcessedTime();
    console.log('Last processed time:', lastProcessed);
    
    // Search for new emails since last run
    const searchQuery = buildSearchQuery(lastProcessed);
    const threads = GmailApp.search(searchQuery, 0, 3);
    
    console.log(`Found ${threads.length} threads to process`);
    
    let processedCount = 0;
    const currentTime = new Date();
    
    for (const thread of threads) {
      const messages = thread.getMessages();
      
      for (const message of messages) {
        // Skip if message is older than our last processed time
        if (message.getDate() <= lastProcessed) {
          continue;
        }
        
        // Skip if message is from me
        if (isFromMe(message)) {
          continue;
        }
        
        // Check if message meets our criteria
        if (shouldProcessMessage(message)) {
          console.log(`Processing message: ${message.getSubject()}`);
          processMessage(message);
          processedCount++;
        }
      }
    }
    
    // Update the last processed timestamp
    setLastProcessedTime(currentTime);
    
    console.log(`Successfully processed ${processedCount} messages`);
    
  } catch (error) {
    console.error('Error in processNewEmails:', error);
    // Continue running even if there's an error
  }
}

/**
 * Check if a message should be processed based on criteria
 */
function shouldProcessMessage(message) {
  const subject = message.getSubject().toLowerCase();
  const body = message.getPlainBody().toLowerCase();
  const to = message.getTo().toLowerCase();
  const cc = message.getCc().toLowerCase();
  
  // Check if directly addressed to me (you'll need to update with your email)
  const myEmail = Session.getActiveUser().getEmail().toLowerCase();
  const directlyAddressed = to.includes(myEmail) || cc.includes(myEmail);
  
  // Check if contains target names
  const targetNames = ['andy', 'andrew', 'casey'];
  const containsTargetName = targetNames.some(name => 
    subject.includes(name) || body.includes(name)
  );
  
  return directlyAddressed || containsTargetName;
}

/**
 * Check if message is from the current user
 */
function isFromMe(message) {
  const myEmail = Session.getActiveUser().getEmail().toLowerCase();
  return message.getFrom().toLowerCase().includes(myEmail);
}

/**
 * Process a single message by generating drafts
 */
async function processMessage(message) {
  try {
    // Get message details
    const subject = message.getSubject();
    const body = message.getPlainBody();
    const from = message.getFrom();
    
    // Generate drafts using Gemini
    const drafts = await generateDraftsWithGemini(subject, body, from);
    
    if (drafts && drafts.length > 0) {
      // Create draft replies in Gmail
      const thread = message.getThread();
      
      for (let i = 0; i < drafts.length; i++) {
        const draftSubject = subject.startsWith('Re:') ? subject : `Re: ${subject}`;
        
        // Create draft
        const draft = thread.createDraftReply(drafts[i]);
        console.log(`Created draft ${i + 1} for: ${subject}`);
      }
    }
    
  } catch (error) {
    console.error('Error processing message:', error);
  }
}

/**
 * Generate draft responses using Gemini AI
 */
async function generateDraftsWithGemini(subject, body, from) {
  try {
    const prompt = buildGeminiPrompt(subject, body, from);
    
    const payload = {
      contents: [{
        parts: [{
          text: prompt
        }]
      }]
    };
    
    const options = {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-goog-api-key': GEMINI_API_KEY
      },
      payload: JSON.stringify(payload)
    };
    
    const response = UrlFetchApp.fetch(GEMINI_API_URL, options);
    const responseData = JSON.parse(response.getContentText());
    
    if (responseData.candidates && responseData.candidates.length > 0) {
      const generatedText = responseData.candidates[0].content.parts[0].text;
      return parseGeminiResponse(generatedText);
    }
    
    return [];
    
  } catch (error) {
    console.error('Error calling Gemini API:', error);
    return [];
  }
}

/**
 * Build the prompt for Gemini AI
 */
function buildGeminiPrompt(subject, body, from) {
  return `You are Andy Casey, an astrophysics professor. Please generate draft email responses to the following email.

EMAIL SUBJECT: ${subject}
FROM: ${from}
EMAIL BODY: ${body}

INSTRUCTIONS:
Provide TWO draft responses:
   - DRAFT 1: A positive response that concisely addresses what Andy Casey is being asked, without setting strict timescales
   - DRAFT 2: An extremely polite "no" response

Format your response as:
DRAFT 1:
[positive response text]

DRAFT 2:
[polite no response text]

Keep responses professional, concise, and appropriate for academic correspondence.`;
}

/**
 * Parse the response from Gemini to extract drafts
 */
function parseGeminiResponse(responseText) {
  const drafts = [];
  
  // Look for DRAFT 1: and DRAFT 2: patterns
  const draft1Match = responseText.match(/DRAFT 1:\s*([\s\S]*?)(?=DRAFT 2:|$)/i);
  const draft2Match = responseText.match(/DRAFT 2:\s*([\s\S]*?)$/i);
  
  if (draft1Match) {
    drafts.push(draft1Match[1].trim());
  }
  
  if (draft2Match) {
    drafts.push(draft2Match[1].trim());
  }
  
  // If no clear draft separation, treat the whole response as a single draft
  if (drafts.length === 0) {
    drafts.push(responseText.trim());
  }
  
  return drafts;
}

/**
 * Build search query for Gmail
 */
function buildSearchQuery(lastProcessed) {
  const yesterday = new Date(Date.now() - 24 * 60 * 60 * 1000);
  const searchDate = lastProcessed > yesterday ? lastProcessed : yesterday;
  
  const year = searchDate.getFullYear();
  const month = searchDate.getMonth() + 1;
  const day = searchDate.getDate();
  const dateString = `${year}/${month}/${day}`;
  
  return `in:inbox after:${dateString} -from:me`;
}

/**
 * Get the last processed timestamp from Properties Service
 */
function getLastProcessedTime() {
  const properties = PropertiesService.getScriptProperties();
  const lastProcessedStr = properties.getProperty('lastProcessedTime');
  
  if (lastProcessedStr) {
    return new Date(lastProcessedStr);
  }
  
  // Default to 12 hour ago if no previous run
  return new Date(Date.now() - 12 * 60 * 60 * 1000);
}

/**
 * Save the last processed timestamp
 */
function setLastProcessedTime(timestamp) {
  const properties = PropertiesService.getScriptProperties();
  properties.setProperty('lastProcessedTime', timestamp.toISOString());
}

/**
 * Delete old drafts
 */

function deleteOldDrafts(days=28) {
  try {
    // Calculate the cutoff date
    const fourWeeksAgo = new Date();
    fourWeeksAgo.setDate(fourWeeksAgo.getDate() - days);
    
    console.log(`Deleting drafts older than: ${fourWeeksAgo.toDateString()}`);
    
    // Get all draft threads
    const draftThreads = GmailApp.getDraftMessages();
    
    let deletedCount = 0;
    let totalDrafts = draftThreads.length;
    
    console.log(`Found ${totalDrafts} total drafts to check`);
    
    // Process each draft
    for (let i = 0; i < draftThreads.length; i++) {
      const draft = draftThreads[i];
      const draftDate = draft.getDate();
      
      // Check if draft is older than the cutoff
      if (draftDate < fourWeeksAgo) {
        try {
          // Get the draft object and delete it
          const gmailDraft = draft.getDraft();
          gmailDraft.delete();
          deletedCount++;
          
          console.log(`Deleted draft from ${draftDate.toDateString()}: "${draft.getSubject() || '(no subject)'}"`);
        } catch (deleteError) {
          console.error(`Failed to delete draft from ${draftDate.toDateString()}: ${deleteError.message}`);
        }
      }
    }
    
    console.log(`\nSummary:`);
    console.log(`- Total drafts checked: ${totalDrafts}`);
    console.log(`- Drafts deleted: ${deletedCount}`);
    console.log(`- Drafts remaining: ${totalDrafts - deletedCount}`);
    
    // Return summary for potential use in other functions
    return {
      totalDrafts: totalDrafts,
      deletedCount: deletedCount,
      remainingDrafts: totalDrafts - deletedCount,
      cutoffDate: fourWeeksAgo
    };
    
  } catch (error) {
    console.error('Error in deleteOldDrafts function:', error.message);
    throw error;
  }
}

// Optional: Function to run a dry run (preview what would be deleted without actually deleting)
function previewOldDrafts(days=28) {
  try {
    const fourWeeksAgo = new Date();
    fourWeeksAgo.setDate(fourWeeksAgo.getDate() - days);
    
    console.log(`Previewing drafts that would be deleted (older than: ${fourWeeksAgo.toDateString()})`);
    
    const draftThreads = GmailApp.getDraftMessages();
    let wouldDeleteCount = 0;
    
    console.log(`Checking ${draftThreads.length} total drafts...\n`);
    
    for (let i = 0; i < draftThreads.length; i++) {
      const draft = draftThreads[i];
      const draftDate = draft.getDate();
      
      if (draftDate < fourWeeksAgo) {
        wouldDeleteCount++;
        const subject = draft.getSubject() || '(no subject)';
        const snippet = draft.getPlainBody().substring(0, 50) + '...';
        
        console.log(`WOULD DELETE - Date: ${draftDate.toDateString()}`);
        console.log(`Subject: "${subject}"`);
        console.log(`Preview: ${snippet}\n`);
      }
    }
    
    console.log(`Summary: ${wouldDeleteCount} drafts would be deleted out of ${draftThreads.length} total drafts`);
    
    return wouldDeleteCount;
    
  } catch (error) {
    console.error('Error in previewOldDrafts function:', error.message);
    throw error;
  }
}

/**
 * Setup function to create the time-based trigger
 * Run this once to set up the automation
 */
function setupTrigger() {
  // Delete existing triggers for this function
  const triggers = ScriptApp.getProjectTriggers();
  triggers.forEach(trigger => {
    if (trigger.getHandlerFunction() === 'processNewEmails') {
      ScriptApp.deleteTrigger(trigger);
    }
  });
  
  // Create new trigger to run every minute
  ScriptApp.newTrigger('processNewEmails')
    .timeBased()
    .everyMinutes(5)
    .create();

  ScriptApp.newTrigger('deleteOldDrafts')
    .timeBased()
    .everyDays(1)
    .create();
    
  console.log('Triggers created successfully');
}



/**
 * Function to test the script manually
 */
function testScript() {
  console.log('Running test...');
  processNewEmails();
}
```

Let me know if you find this helpful!