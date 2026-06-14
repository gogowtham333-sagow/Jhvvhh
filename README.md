import React, { useState, useMemo } from 'react';
import { 
  Search, Plus, User, Phone, Mail, Trash2, Edit2, X,
  Briefcase, Home, Heart, Sparkles, MessageSquare, Loader2, Copy, Wand2
} from 'lucide-react';

const INITIAL_CONTACTS = [
  { id: '1', name: 'Alice Smith', email: 'alice.smith@example.com', phone: '(555) 123-4567', category: 'Work' },
  { id: '2', name: 'Bob Jones', email: 'bob.jones@example.com', phone: '(555) 987-6543', category: 'Family' },
  { id: '3', name: 'Charlie Brown', email: 'charlie.b@example.com', phone: '(555) 555-5555', category: 'Personal' },
  { id: '4', name: 'Diana Prince', email: 'diana@example.com', phone: '(555) 888-9999', category: 'Work' },
];

const API_KEY = ""; // Put your Gemini API key here if running locally or on Vercel

export default function App() {
  const [contacts, setContacts] = useState(INITIAL_CONTACTS);
  const [searchQuery, setSearchQuery] = useState('');
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [editingContact, setEditingContact] = useState(null);

  // AI Features State
  const [isExtracting, setIsExtracting] = useState(false);
  const [magicText, setMagicText] = useState('');
  
  const [draftModalOpen, setDraftModalOpen] = useState(false);
  const [draftContact, setDraftContact] = useState(null);
  const [draftContext, setDraftContext] = useState('');
  const [draftResult, setDraftResult] = useState('');
  const [isDrafting, setIsDrafting] = useState(false);
  const [copiedState, setCopiedState] = useState(false);

  // Form State
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    phone: '',
    category: 'Personal'
  });

  // Filter contacts based on search query
  const filteredContacts = useMemo(() => {
    return contacts.filter(contact => 
      contact.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
      contact.email.toLowerCase().includes(searchQuery.toLowerCase()) ||
      contact.phone.includes(searchQuery)
    );
  }, [contacts, searchQuery]);

  // --- AI Helper Functions ---
  const fetchWithRetry = async (url, options, retries = 5) => {
    const delays = [1000, 2000, 4000, 8000, 16000];
    for (let i = 0; i < retries; i++) {
      try {
        const res = await fetch(url, options);
        if (!res.ok) throw new Error(`HTTP error! status: ${res.status}`);
        return await res.json();
      } catch (err) {
        if (i === retries - 1) throw err;
        await new Promise(resolve => setTimeout(resolve, delays[i]));
      }
    }
  };

  const handleMagicImport = async () => {
    if (!magicText.trim()) return;
    setIsExtracting(true);
    try {
      const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${API_KEY}`;
      const payload = {
        contents: [{ parts: [{ text: `Extract contact details from this text: "${magicText}"` }] }],
        systemInstruction: { parts: [{ text: "You are a helpful assistant. Extract name, email, phone, and categorize as 'Work', 'Personal', or 'Family'. Respond strictly in JSON format." }] },
        generationConfig: {
          responseMimeType: "application/json",
          responseSchema: {
            type: "OBJECT",
            properties: {
              name: { type: "STRING" },
              email: { type: "STRING" },
              phone: { type: "STRING" },
              category: { type: "STRING", enum: ["Work", "Personal", "Family"] }
            },
            required: ["name"]
          }
        }
      };
      const data = await fetchWithRetry(url, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload) });
      const resultText = data.candidates?.[0]?.content?.parts?.[0]?.text;
      if (resultText) {
        const parsed = JSON.parse(resultText);
        setFormData(prev => ({
          ...prev,
          name: parsed.name || prev.name,
          email: parsed.email || prev.email,
          phone: parsed.phone || prev.phone,
          category: parsed.category || prev.category
        }));
        setMagicText('');
      }
    } catch (error) {
      console.error("Extraction failed", error);
    } finally {
      setIsExtracting(false);
    }
  };

  const handleDraftMessage = async () => {
    if (!draftContext.trim() || !draftContact) return;
    setIsDrafting(true);
    setDraftResult('');
    try {
      const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${API_KEY}`;
      const prompt = `Write a message to ${draftContact.name} (Context: They are a ${draftContact.category} contact. Email: ${draftContact.email}). The message should be about: ${draftContext}`;
      const payload = {
        contents: [{ parts: [{ text: prompt }] }],
        systemInstruction: { parts: [{ text: "You are a helpful assistant that drafts polite, engaging, and context-appropriate messages. Output ONLY the message content without any conversational filler." }] }
      };
      const data = await fetchWithRetry(url, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload) });
      const resultText = data.candidates?.[0]?.content?.parts?.[0]?.text;
      if (resultText) {
        setDraftResult(resultText);
      }
    } catch (error) {
      setDraftResult("Sorry, I couldn't generate a message right now. Please try again.");
    } finally {
      setIsDrafting(false);
    }
  };

  const copyDraftToClipboard = () => {
    const textArea = document.createElement("textarea");
    textArea.value = draftResult;
    document.body.appendChild(textArea);
    textArea.select();
    document.execCommand("copy");
    document.body.removeChild(textArea);
    setCopiedState(true);
    setTimeout(() => setCopiedState(false), 2000);
  };
  // --- End AI Helper Functions ---

  const handleOpenModal = (contact = null) => {
    if (contact) {
      setEditingContact(contact);
      setFormData(contact);
    } else {
      setEditingContact(null);
      setFormData({ name: '', email: '', phone: '', category: 'Personal' });
    }
    setIsModalOpen(true);
  };

  const handleCloseModal = () => {
    setIsModalOpen(false);
    setEditingContact(null);
    setMagicText('');
  };

  const handleInputChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({ ...prev, [name]: value }));
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    if (!formData.name.trim()) return;

    if (editingContact) {
      setContacts(contacts.map(c => c.id === editingContact.id ? { ...formData, id: c.id } : c));
    } else {
      const newContact = {
        ...formData,
        id: Date.now().toString()
      };
      setContacts([...contacts, newContact]);
    }
    handleCloseModal();
  };

  const handleDelete = (id) => {
    setContacts(contacts.filter(c => c.id !== id));
  };

  const getCategoryIcon = (category) => {
    switch (category) {
      case 'Work': return <Briefcase size={16} />;
      case 'Family': return <Home size={16} />;
      default: return <Heart size={16} />;
    }
  };

  const getCategoryColor = (category) => {
    switch (category) {
      case 'Work': return 'bg-blue-100 text-blue-700 border-blue-200';
      case 'Family': return 'bg-green-100 text-green-700 border-green-200';
      default: return 'bg-purple-100 text-purple-700 border-purple-200';
    }
  };

  return (
    <div className="min-h-screen bg-slate-50 text-slate-800 font-sans">
      <header className="bg-white border-b border-slate-200 sticky top-0 z-10">
        <div className="max-w-5xl mx-auto px-4 py-4 md:py-6 flex flex-col md:flex-row md:items-center justify-between gap-4">
          <div>
            <h1 className="text-2xl font-bold text-slate-900 flex items-center gap-2">
              <User className="text-indigo-600" />
              My Contacts
            </h1>
            <p className="text-sm text-slate-500 mt-1">Manage and organize your connections</p>
          </div>
          
          <div className="flex items-center gap-3 w-full md:w-auto">
            <div className="relative flex-1 md:w-64">
              <Search className="absolute left-3 top-1/2 -translate-y-1/2 text-slate-400" size={18} />
              <input
                type="text"
                placeholder="Search contacts..."
                value={searchQuery}
                onChange={(e) => setSearchQuery(e.target.value)}
                className="w-full pl-10 pr-4 py-2 bg-slate-100 border-transparent focus:bg-white focus:border-indigo-500 focus:ring-2 focus:ring-indigo-200 rounded-lg outline-none transition-all"
              />
            </div>
            <button
              onClick={() => handleOpenModal()}
              className="bg-indigo-600 hover:bg-indigo-700 text-white p-2 md:px-4 md:py-2 rounded-lg flex items-center gap-2 transition-colors shadow-sm"
            >
              <Plus size={20} />
              <span className="hidden md:inline font-medium">Add Contact</span>
            </button>
          </div>
        </div>
      </header>

      <main className="max-w-5xl mx-auto px-4 py-8">
        {filteredContacts.length === 0 ? (
          <div className="text-center py-20 bg-white rounded-2xl border border-dashed border-slate-300">
            <div className="bg-slate-100 w-16 h-16 rounded-full flex items-center justify-center mx-auto mb-4 text-slate-400">
              <User size={32} />
            </div>
            <h3 className="text-lg font-medium text-slate-900 mb-1">No contacts found</h3>
            <p className="text-slate-500 max-w-sm mx-auto">
              {searchQuery ? "We couldn't find anyone matching your search." : "Your contact list is empty. Add a new contact to get started."}
            </p>
            {searchQuery && (
              <button 
                onClick={() => setSearchQuery('')}
                className="mt-4 text-indigo-600 font-medium hover:underline"
              >
                Clear search
              </button>
            )}
          </div>
        ) : (
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 md:gap-6">
            {filteredContacts.map(contact => (
              <div 
                key={contact.id} 
                className="bg-white rounded-2xl p-5 border border-slate-200 shadow-sm hover:shadow-md transition-shadow group relative overflow-hidden"
              >
                <div className="flex justify-between items-start mb-4">
                  <div className="flex items-center gap-3">
                    <div className="w-12 h-12 bg-gradient-to-br from-indigo-100 to-purple-100 text-indigo-700 rounded-full flex items-center justify-center font-bold text-lg">
                      {contact.name.charAt(0).toUpperCase()}
                    </div>
                    <div>
                      <h3 className="font-semibold text-slate-900 text-lg leading-tight">{contact.name}</h3>
                      <span className={`inline-flex items-center gap-1 px-2 py-0.5 rounded-full text-xs font-medium border mt-1 ${getCategoryColor(contact.category)}`}>
                        {getCategoryIcon(contact.category)}
                        {contact.category}
                      </span>
                    </div>
                  </div>
                  
                  <div className="flex opacity-100 md:opacity-0 group-hover:opacity-100 transition-opacity gap-1">
                    <button 
                      onClick={() => {
                        setDraftContact(contact);
                        setDraftContext('');
                        setDraftResult('');
                        setDraftModalOpen(true);
                      }}
                      className="p-1.5 text-slate-400 hover:text-amber-600 hover:bg-amber-50 rounded-md transition-colors"
                      title="✨ Draft Message"
                    >
                      <MessageSquare size={16} />
                    </button>
                    <button 
                      onClick={() => handleOpenModal(contact)}
                      className="p-1.5 text-slate-400 hover:text-indigo-600 hover:bg-indigo-50 rounded-md transition-colors"
                      title="Edit"
                    >
                      <Edit2 size={16} />
                    </button>
                    <button 
                      onClick={() => handleDelete(contact.id)}
                      className="p-1.5 text-slate-400 hover:text-red-600 hover:bg-red-50 rounded-md transition-colors"
                      title="Delete"
                    >
                      <Trash2 size={16} />
                    </button>
                  </div>
                </div>

                <div className="space-y-2.5">
                  {contact.phone && (
                    <div className="flex items-center gap-2.5 text-slate-600 text-sm">
                      <Phone size={16} className="text-slate-400" />
                      <a href={`tel:${contact.phone}`} className="hover:text-indigo-600 transition-colors">{contact.phone}</a>
                    </div>
                  )}
                  {contact.email && (
                    <div className="flex items-center gap-2.5 text-slate-600 text-sm">
                      <Mail size={16} className="text-slate-400" />
                      <a href={`mailto:${contact.email}`} className="hover:text-indigo-600 transition-colors truncate">{contact.email}</a>
                    </div>
                  )}
                </div>
              </div>
            ))}
          </div>
        )}
      </main>

      {/* Add/Edit Modal */}
      {isModalOpen && (
        <div className="fixed inset-0 bg-slate-900/50 backdrop-blur-sm z-50 flex items-center justify-center p-4">
          <div className="bg-white rounded-2xl shadow-xl w-full max-w-md overflow-hidden flex flex-col max-h-[90vh]">
            <div className="px-6 py-4 border-b border-slate-100 flex justify-between items-center bg-slate-50/50">
              <h2 className="text-xl font-semibold text-slate-900">
                {editingContact ? 'Edit Contact' : 'Add New Contact'}
              </h2>
              <button 
                onClick={handleCloseModal}
                className="text-slate-400 hover:text-slate-600 p-1 rounded-full hover:bg-slate-100 transition-colors"
              >
                <X size={20} />
              </button>
            </div>
            
            <form onSubmit={handleSubmit} className="p-6 overflow-y-auto">
              {!editingContact && (
                <div className="mb-6 bg-gradient-to-br from-indigo-50 to-purple-50 p-4 rounded-xl border border-indigo-100">
                  <label className="flex items-center gap-2 text-sm font-semibold text-indigo-900 mb-2">
                    <Sparkles size={16} className="text-indigo-600" />
                    ✨ Magic Import
                  </label>
                  <p className="text-xs text-indigo-700 mb-3">Paste a signature or text block, and we'll extract the details!</p>
                  <div className="flex gap-2">
                    <textarea
                      value={magicText}
                      onChange={(e) => setMagicText(e.target.value)}
                      placeholder="E.g. John Doe, Developer. 555-0198, john@example.com"
                      className="w-full px-3 py-2 text-sm border border-indigo-200 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500 bg-white resize-none"
                      rows="2"
                    />
                    <button
                      type="button"
                      onClick={handleMagicImport}
                      disabled={isExtracting || !magicText.trim()}
                      className="px-3 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700 disabled:opacity-50 disabled:cursor-not-allowed flex items-center justify-center transition-colors shadow-sm"
                      title="Extract Info"
                    >
                      {isExtracting ? <Loader2 size={18} className="animate-spin" /> : <Wand2 size={18} />}
                    </button>
                  </div>
                </div>
              )}

              <div className="space-y-4">
                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">Full Name *</label>
                  <input
                    type="text"
                    name="name"
                    required
                    value={formData.name}
                    onChange={handleInputChange}
                    className="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500 transition-shadow"
                    placeholder="Jane Doe"
                  />
                </div>
                
                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">Email Address</label>
                  <input
                    type="email"
                    name="email"
                    value={formData.email}
                    onChange={handleInputChange}
                    className="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500 transition-shadow"
                    placeholder="jane@example.com"
                  />
                </div>
                
                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">Phone Number</label>
                  <input
                    type="tel"
                    name="phone"
                    value={formData.phone}
                    onChange={handleInputChange}
                    className="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500 transition-shadow"
                    placeholder="(555) 000-0000"
                  />
                </div>
                
                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">Category</label>
                  <div className="grid grid-cols-3 gap-2 mt-2">
                    {['Personal', 'Work', 'Family'].map(category => (
                      <button
                        key={category}
                        type="button"
                        onClick={() => setFormData(prev => ({ ...prev, category }))}
                        className={`py-2 px-3 border rounded-lg text-sm font-medium flex items-center justify-center gap-1.5 transition-all
                          ${formData.category === category 
                            ? 'bg-indigo-50 border-indigo-200 text-indigo-700 ring-1 ring-indigo-200' 
                            : 'bg-white border-slate-200 text-slate-600 hover:bg-slate-50'
                          }`}
                      >
                        {getCategoryIcon(category)}
                        {category}
                      </button>
                    ))}
                  </div>
                </div>
              </div>
              
              <div className="mt-8 flex gap-3">
                <button
                  type="button"
                  onClick={handleCloseModal}
                  className="flex-1 px-4 py-2.5 border border-slate-300 text-slate-700 rounded-lg font-medium hover:bg-slate-50 transition-colors"
                >
                  Cancel
                </button>
                <button
                  type="submit"
                  className="flex-1 px-4 py-2.5 bg-indigo-600 text-white rounded-lg font-medium hover:bg-indigo-700 transition-colors shadow-sm"
                >
                  {editingContact ? 'Save Changes' : 'Add Contact'}
                </button>
              </div>
            </form>
          </div>
        </div>
      )}

      {/* Draft Message Modal */}
      {draftModalOpen &
