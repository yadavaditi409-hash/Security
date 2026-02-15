# Security

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

  // 1. Fetch Tabs First
  useEffect(() => {
    const fetchTabs = async () => {
      try {
        const res = await axios.get(`https://localhost:7236/api/Security/tabs/${securityType}`);
        setTabs(res.data);
        if (res.data.length > 0) setSelectedTab(res.data[0].tabId);
      } catch (err) {
        setError("Could not load security metadata.");
      }
    };
    fetchTabs();
  }, [securityType]);

  // 2. Load Attribute & Data according to (sectype, id, tabid)
  useEffect(() => {
    if (selectedTab) {
      const fetchTabData = async () => {
        setLoading(true);
        try {
          // Fetch attributes and security values for the specific tab
          const [attrRes, dataRes] = await Promise.all([
            axios.get(`https://localhost:7236/api/security/attributes/${securityType}/${selectedTab}`),
            axios.get(`https://localhost:7236/api/security/viewsecuritybyid/${securityType}/${id}/${selectedTab}`)
          ]);
          setAttributes(attrRes.data);
          setTabData(dataRes.data);
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

    // Update local display state
    setTabData(prev => ({ ...prev, [cleanKey]: val }));
    
    // Add to patchData object for the final submit
    setPatchData(prev => ({ ...prev, [cleanKey]: val }));
  };

  const handleUpdate = async () => {
    if (Object.keys(patchData).length === 0) {
      setIsEditing(false);
      return;
    }

    try {
      setLoading(true);
      // Patching the collected changed attributes to the DB
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
      setError("Update failed. Please verify the connection or data format.");
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="p-8 font-sans max-w-6xl mx-auto">
      <div id="success-banner" className="hidden fixed top-4 right-4 bg-green-600 text-white px-6 py-3 rounded shadow-lg z-50">
        Changes saved successfully!
      </div>

      <div className="flex justify-between items-center mb-6">
        <button onClick={() => navigate(-1)} className="text-blue-600 hover:underline flex items-center gap-2">
          <span>←</span> Back to list
        </button>
        <div className="flex gap-3">
          {!isEditing ? (
            <button 
              onClick={() => setIsEditing(true)} 
              className="bg-blue-600 hover:bg-blue-700 text-white px-6 py-2 rounded-lg font-bold shadow transition-all"
            >
              Edit Security
            </button>
          ) : (
            <>
              <button 
                onClick={() => { setIsEditing(false); setPatchData({}); }} 
                className="bg-gray-100 hover:bg-gray-200 text-gray-700 px-6 py-2 rounded-lg font-bold transition-all"
              >
                Cancel
              </button>
              <button 
                onClick={handleUpdate} 
                className="bg-green-600 hover:bg-green-700 text-white px-6 py-2 rounded-lg font-bold shadow transition-all"
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
        <p className="text-gray-400 text-sm font-mono">{securityType.toUpperCase()} ID: {id}</p>
      </div>

      {error && (
        <div className="bg-red-50 border-l-4 border-red-500 text-red-700 p-4 mb-6 rounded-r">
          <p>{error}</p>
        </div>
      )}

      {/* Tabs Navigation */}
      <div className="flex gap-1 mb-6 border-b overflow-x-auto bg-gray-50 p-1 rounded-t-lg">
        {tabs.map(t => (
          <button 
            key={t.tabId} 
            onClick={() => { setSelectedTab(t.tabId); setPatchData({}); setIsEditing(false); }}
            className={`px-6 py-3 text-sm font-bold rounded-lg transition-all ${
              selectedTab === t.tabId 
                ? 'bg-white text-blue-600 shadow-sm' 
                : 'text-gray-500 hover:text-gray-700 hover:bg-gray-100'
            }`}
          >
            {t.tabName}
          </button>
        ))}
      </div>

      {/* Attributes Content Area */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-8 bg-white p-8 border rounded-b-xl shadow-sm min-h-[300px]">
        {loading ? (
          <div className="col-span-full text-center text-gray-400">Loading tab data...</div>
        ) : attributes.length > 0 ? attributes.map(attr => {
          const cleanKey = attr.replace(/\s+/g, "");
          const value = tabData[cleanKey] ?? tabData[cleanKey.charAt(0).toLowerCase() + cleanKey.slice(1)] ?? "";
          const type = getInputType(attr);

          return (
            <div key={attr} className="flex flex-col">
              <label className="text-[10px] font-black text-gray-400 uppercase tracking-widest mb-2">
                {attr}
              </label>
              {isEditing ? (
                <input 
                  type={type}
                  value={type !== "checkbox" ? (type === 'date' && value ? value.split('T')[0] : value) : undefined}
                  checked={type === "checkbox" ? !!value : undefined}
                  onChange={(e) => handleChange(e, attr)}
                  className="p-3 border-2 border-gray-100 rounded-lg focus:border-blue-500 outline-none w-full bg-blue-50/20 font-medium text-gray-700"
                />
              ) : (
                <div className="text-base text-gray-800 font-semibold py-1">
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
          <div className="col-span-full flex items-center justify-center text-gray-300 italic">
            No fields found for this section.
          </div>
        )}
      </div>
    </div>
  );
};
