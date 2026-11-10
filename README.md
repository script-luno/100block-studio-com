(async function() {
    try {
        console.log("📦 ZIP圧縮ライブラリをロード中...");
        
        // 1. CDNから JSZip ライブラリを動的に読み込む
        if (typeof JSZip === 'undefined') {
            await new Promise((resolve, reject) => {
                const script = document.createElement('script');
                script.src = 'https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js';
                script.onload = resolve;
                script.onerror = reject;
                document.head.appendChild(script);
            });
        }
        console.log("✅ ライブラリのロード完了。プロジェクトデータを解析中...");

        // 2. プロジェクトIDの取得
        const match = window.location.href.match(/\/projects\/(\d+)/);
        if (!match) {
            console.error("❌ Scratchのプロジェクトページ（/projects/ID）で実行してください。");
            return;
        }
        const projectId = match[1];

        // 3. メタデータと project.json の取得
        const metaRes = await fetch(`https://api.scratch.mit.edu/projects/${projectId}`);
        const metaData = await metaRes.json();

        const jsonRes = await fetch(`https://projects.scratch.mit.edu/${projectId}?token=${metaData.project_token}`);
        const jsonText = await jsonRes.text();
        const projectJson = JSON.parse(jsonText);

        // 4. JSZipのインスタンスを作成し、project.json を追加
        const zip = new JSZip();
        zip.file("project.json", jsonText);

        // 5. すべてのアセットのハッシュを収集（重複排除）
        const assetMap = new Map();
        projectJson.targets.forEach(target => {
            target.costumes?.forEach(c => { if (c.md5ext) assetMap.set(c.md5ext, c.md5ext); });
            target.sounds?.forEach(s => { if (s.md5ext) assetMap.set(s.md5ext, s.md5ext); });
        });

        console.log(`📦 対象アセット数: ${assetMap.size} 個。ファイルをダウンロードしてZIPに追加中...`);

        // 6. すべてのアセットを並行して取得し、ZIPに詰めていく
        let successCount = 0;
        let failCount = 0;

        const promises = Array.from(assetMap.keys()).map(async (hash) => {
            try {
                const assetRes = await fetch(`https://assets.scratch.mit.edu/internalapi/asset/${hash}/get/`);
                if (assetRes.ok) {
                    const arrayBuffer = await assetRes.arrayBuffer();
                    zip.file(hash, arrayBuffer);
                    successCount++;
                } else {
                    failCount++;
                }
            } catch (e) {
                failCount++;
            }
        });

        await Promise.all(promises);
        console.log(`🔄 すべての取得が完了 (${successCount}成功, ${failCount}失敗)。ZIP圧縮を実行中...`);

        // 7. メモリ上でZIPとして固め、そのファイルサイズを取得
        const content = await zip.generateAsync({
            type: "blob",
            compression: "DEFLATE",
            compressionOptions: { level: 6 }
        });

        const sb3Bytes = content.size;
        const kb = (sb3Bytes / 1024).toFixed(2);
        const mb = (sb3Bytes / (1024 * 1024)).toFixed(2);

        // 8. 結果表示
        console.log("----------------------------------------");
        console.log(`🎉 【ZIP実測完了】 ${metaData.title}`);
        console.log(`🔹 .sb3 換算の総バイト数: ${sb3Bytes.toLocaleString()} bytes`);
        console.log(`🔹 KB 換算:               ${kb} KB`);
        if (mb >= 1) {
            console.log(`🔹 MB 換算:               ${mb} MB`);
        }
        console.log("----------------------------------------");

    } catch (err) {
        console.error("❌ エラーが発生しました:", err);
    }
})();
