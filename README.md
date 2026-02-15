const DisplayingASecurity = () => {
  const { securityType, id } = useParams();
  const navigate = useNavigate();
  const [tabs, setTabs] = useState([]);
  const [selectedTab, setSelectedTab] = useState(null);
  const [attributes, setAttributes] = useState([]);
  const [tabData, setTabData] = useState({});
  const [patchData, setPatchData] = useState({});
  const [isEditing, setIsEditing] = useState(false);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const getInputType = (attr) => {
    const low = attr.toLowerCase();
    if (low.includes("date")) return "date";
    if (low.startsWith("is ") || low.startsWith("has ")) return "checkbox";
    if (low.includes("price") || low.includes("rate") || low.includes("yield")) return "number";
    return "text";
  };

  // 1. Fetch tabs first to get a valid tabId
  useEffect(() => {
    const fetchTabs = async () => {
      try {
        const res = await axios.get(`https://localhost:7236/api/Security/tabs/${securityType}`);
        setTabs(res.data);
        // Default to first tab (tabId 1) if available
        if (res.data.length > 0) setSelectedTab(res.data[0].tabId);
      } catch (err) {
        setError("Could not load security tabs.");
      }
    };
    fetchTabs();
  }, [securityType]);

  // 2. Fetch data based on securityType, id, and selectedTab
  useEffect(() => {
    if (selectedTab && id) {
      const fetchTabData = async () => {
        setLoading(true);
        setError(null);
        try {
          const [attrRes, dataRes] = await Promise.all([
            axios.get(`https://localhost:7236/api/security/attributes/${securityType}/${selectedTab}`),
            axios.get(`https://localhost:7236/api/security/viewsecuritybyid/${securityType}/${id}/${selectedTab}`)
          ]);
          setAttributes(attrRes.data);
          setTabData(dataRes.data || {});
          // Clear patch data when changing tabs
          setPatchData({});
          setIsEditing(false);
        } catch (e) {
          setError("Failed to load data for this tab.");
        } finally {
          setLoading(false);
        }
      };
      fetchTabData();
    }
  }, [selectedTab, securityType, id]);

  const handleChange = (e, attr) => {
    const cleanKey = attr.replace(/\s+/g, "");
    const type = getInputType(attr);
    let val = type === "checkbox" ? e.target.checked : e.target.value;
    
    if (type === "number" && val !== "") val = parseFloat(val);

    setTabData(prev => ({ ...prev, [cleanKey]: val }));
    setPatchData(prev => ({ ...prev, [cleanKey]: val }));
  };

  const handleUpdate = async () => {
    if (Object.keys(patchData).length === 0) {
      setIsEditing(false);
      return;
    }

    try {
      setLoading(true);
      await axios.patch(`https://localhost:7236/api/${securityType}/${id}`, patchData);
      
      setPatchData({});
      setIsEditing(false);
      setError(null);
      
      const successMsg = document.getElementById('success-banner');
      if (successMsg) {
        successMsg.classList.remove('hidden');
        setTimeout(() => successMsg.classList.add('hidden'), 3000);
      }
    } catch (err) {
      setError("Update failed. Check if all required fields are present.");
    } finally {
      setLoading(false);
    }
  };

  if (!id) return <div className="p-10 text-center text-red-500">Error: No Security ID provided.</div>;

  return (
    <div className="p-8 font-sans max-w-6xl mx-auto">
      <div id="success-banner" className="hidden fixed top-4 right-4 bg-green-600 text-white px-6 py-3 rounded shadow-lg z-50">
        Changes saved successfully!
      </div>

      <div className="flex justify-between items-center mb-6">
        <button onClick={() => navigate(-1)} className="text-blue-600 hover:underline flex items-center gap-2 font-medium">
          ← Back to List
        </button>
        <div className="flex gap-3">
          {!isEditing ? (
            <button 
              onClick={() => setIsEditing(true)} 
              className="bg-blue-600 hover:bg-blue-700 text-white px-6 py-2 rounded-lg font-bold shadow-sm"
            >
              Edit Details
            </button>
          ) : (
            <>
              <button 
                onClick={() => { setIsEditing(false); setPatchData({}); }} 
                className="bg-gray-100 text-gray-700 px-6 py-2 rounded-lg font-bold"
              >
                Cancel
              </button>
              <button 
                onClick={handleUpdate} 
                className="bg-green-600 hover:bg-green-700 text-white px-6 py-2 rounded-lg font-bold shadow-sm"
              >
                {loading ? "Saving..." : "Submit Changes"}
              </button>
            </>
          )}
        </div>
      </div>

      <div className="bg-white rounded-xl shadow-sm border p-6 mb-6">
        <h2 className="text-3xl font-extrabold text-gray-800 mb-1">
          {tabData.SecurityName || tabData.securityName || 'Security Details'}
        </h2>
        <p className="text-gray-400 text-sm font-mono tracking-tight">System ID: {id}</p>
      </div>

      {error && <div className="bg-red-50 border-l-4 border-red-500 text-red-700 p-4 mb-6 rounded shadow-sm">{error}</div>}

      {/* Tabs Navigation */}
      <div className="flex gap-1 mb-6 border-b overflow-x-auto bg-gray-50 p-1 rounded-t-lg">
        {tabs.map(t => (
          <button 
            key={t.tabId} 
            onClick={() => setSelectedTab(t.tabId)}
            className={`px-6 py-3 text-sm font-bold rounded-lg transition-all whitespace-nowrap ${
              selectedTab === t.tabId ? 'bg-white text-blue-600 shadow-sm' : 'text-gray-500 hover:bg-gray-100'
            }`}
          >
            {t.tabName}
          </button>
        ))}
      </div>

      {/* Attributes Content Area */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-8 bg-white p-8 border rounded-b-xl shadow-sm min-h-[300px]">
        {loading ? (
          <div className="col-span-full text-center text-gray-400 py-20">Fetching security details...</div>
        ) : attributes.length > 0 ? attributes.map(attr => {
          const cleanKey = attr.replace(/\s+/g, "");
          const value = tabData[cleanKey] ?? tabData[cleanKey.charAt(0).toLowerCase() + cleanKey.slice(1)] ?? "";
          const type = getInputType(attr);

          return (
            <div key={attr} className="flex flex-col">
              <label className="text-[10px] font-black text-gray-400 uppercase tracking-widest mb-2">{attr}</label>
              {isEditing ? (
                <input 
                  type={type}
                  value={type !== "checkbox" ? (type === 'date' && value ? value.split('T')[0] : value) : undefined}
                  checked={type === "checkbox" ? !!value : undefined}
                  onChange={(e) => handleChange(e, attr)}
                  className="p-3 border-2 border-gray-100 rounded-lg focus:border-blue-500 outline-none w-full bg-blue-50/10"
                />
              ) : (
                <div className="text-base text-gray-800 font-semibold py-1 border-b border-gray-50">
                  {type === "checkbox" ? (
                    <span className={`px-2 py-0.5 rounded text-xs ${value ? 'bg-green-100 text-green-700' : 'bg-gray-100 text-gray-500'}`}>
                      {value ? "✓ Yes" : "✗ No"}
                    </span>
                  ) : 
                   type === "date" && value ? (
                     <span className="text-gray-600">{new Date(value).toLocaleDateString()}</span>
                   ) : 
                   String(value || '—')}
                </div>
              )}
            </div>
          );
        }) : (
          <div className="col-span-full text-center text-gray-300 italic">No attributes found for this section.</div>
        )}
      </div>
    </div>
  );
};
