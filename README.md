# huaiuzo_easyui

这是一个js/css文件压缩工具
        /// <summary>
        /// 读取xml关于js/css需要合并操作的文件
        /// </summary>
        /// <param name="filePath"></param>
        /// <param name="type"></param>
        /// <returns></returns>
        private async Task ReadXml(string filePath, string type = "js")
        {
            _type = type;
            var current = filePath + "\\Config\\bundle.xml";
            if (type != "js")
                current = filePath + "\\Config\\bundle2css.xml";

            var root = XElement.Load(current);
            IEnumerable address = from el in root.Elements("styleBundle")
                select el;

            foreach (var t in address.Cast<XElement>())
            {
                var name = t.Attribute("name").Value;
                var list = new List<string>();
                foreach (var el2 in t.Elements("include"))
                {
                    var attr = el2.Attribute("path").ToString();
                    var start = attr.IndexOf("\"");
                    var len = attr.LastIndexOf(type) - attr.IndexOf("\"");
                    if (len <= 0 || start == -1) continue;

                    var path = attr.Substring(attr.IndexOf("\"") + 1, len + 1);
                    if (type == "css")
                    {
                        path = attr.Substring(attr.IndexOf("\"") + 1, len + 2);
                        if (string.IsNullOrWhiteSpace(path) || path.LastIndexOf("css") < 0)
                            continue;
                    }
                    else
                    {
                        if (string.IsNullOrWhiteSpace(path) || path.LastIndexOf("js") < 0)
                            continue;
                    }

                    list.Add(path);
                }
                if (list.Count > 0)
                    await Zip(name, _filePath, list);
            }
        }

        /// <summary>
        ///     读取公用资源
        /// </summary>
        /// <param name="name"></param>
        /// <param name="dic"></param>
        /// <param name="list"></param>
        /// <returns></returns>
        public List<string> ReadPublic(string name, Dictionary<string, List<string>> dic, List<string> list)
        {
            if (_listConfig.Contains(name))
            {
                dic.Add(name, list);
            }
            else
            {
                var items = new List<string>();
                foreach (var item in dic)
                {
                    if (name.StartsWith("passport_") && item.Key.StartsWith("passport_"))
                    {
                        items.AddRange(item.Value);
                    }
                    else if (name.StartsWith("mall_") && item.Key.StartsWith("mall_"))
                    {
                        items.AddRange(item.Value);
                    }
                    else if (name.StartsWith("topic_") && item.Key.StartsWith("topic_"))
                    {
                        items.AddRange(item.Value);
                    }
                }
                items.AddRange(list);
                return items;
            }
            return list;
        }

        /// <summary>
        ///     压缩
        /// </summary>
        /// <param name="name"></param>
        /// <param name="path"></param>
        /// <param name="fileNames"></param>
        public async Task Zip(string name, string path, List<string> fileNames)
        {
            var listFile = new List<string>();
            foreach (var file in fileNames)
            {
                var directoryInfo = path + "\\" + file.Replace("/", "\\");
                listFile.Add(directoryInfo);
            }

            var origin = string.Join(",", listFile.ToArray());
            string target;
            if (_type == "js")
                target = path + "\\js\\Min\\" + name + ".js";
            else
                target = path + "\\css\\" + name + ".css";
            await Merger(name, origin, target);
        }

        /// <summary>
        ///     关于合并
        /// </summary>
        /// <param name="name"></param>
        /// <param name="origin"></param>
        /// <param name="target"></param>
        private async Task Merger(string name, string origin, string target)
        {
            var t = string.Empty;
            var originList = origin.Split(',');
            for (int i = 0, l = originList.Length; i < l; i++)
            {
                var o = originList[i];
                var file = File.ReadAllText(o, _encode);
                if (_type == "js")
                {
                    var js = new JavaScriptCompressor(file, false, _encode, CultureInfo.CurrentCulture);
                    t += js.Compress(true, true, true, int.MaxValue);
                }
                else
                {
                    t += CssCompressor.Compress(file);
                }
            }
            await SaveFile(t, target);
            if (_type == "js")
                AddNode(name, "/js/Min/" + name + ".js", t);
            else
                AddNode(name, "/css/" + name + ".css", t);
        }
