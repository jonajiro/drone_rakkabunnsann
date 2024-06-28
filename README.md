# drone_rakkabunnsann
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>落下分散</title>
    <style>
        .form-container {
            display: flex;
            flex-direction: column;
            position: absolute;
            top: 40px;
            left: 350px;
            width: 300px;
        }
        .form-container label {
          display: inline-block;
          width:150px;
          vertical-align:top;
            margin-bottom: 10px;
        }
        .form-container button {
            margin-top: 10px;
        }
        .container {
            position: relative;
            width: 120px;
            height: 20px;
            margin: 20px;
        }
        canvas {
            border: 2px solid black;
            position: absolute;
            left: 40px;
            top: 20px;
        }
        .x-axis-label {
            position: absolute;
            left: 40px;
            top:  540px;
            width: 200px;
            text-align: center;
        }
        .y-axis-label {
            position: absolute;
            left: 0;
            top: 270px;
            height: 100px;
            text-align: center;
            transform: rotate(-90deg);
            transform-origin: left top;
        }

    </style>
</head>
<body>

    <div class="form-container">
        <label>質量 (kg):　　　　 <input type="number" id="input_mass" value="0.9" step="0.1"></label>
        <label>機体最大幅 (m):　  <input type="number" id="input_bodylen" value="0.354" step="0.1"></label>
        <label>粘性係数 (Pa・S):　<input type="number" id="input_mu" value="0.00001822" step="0.1"></label>
        <label>空気密度 (kg/m^3): <input type="number" id="input_rho" value="1.205" step="0.1"></label>
        <label>横風の速度 (m/s):  <input type="number" id="input_ws" value="5.0" step="0.1"></label>
        <label>水平方向速度 (m/s):<input type="number" id="input_Vx" value="0.0" step="0.1"></label>
        <label>飛行高度 (m):　    <input type="number" id="input_Y" value="120.0" step="0.1"></label><br />

        <button onclick="updateTrajectory()">更新</button>
        <label>抗力係数 (参考値):  <output type="number" id="result_cd" value="0.0" step="0.1"></label>
        <label>落下時間 (参考値):  <output type="number" id="result_t" value="0.0" step="0.1"></label>
        <label>最終速度 (参考値):  <output type="number" id="result_vel" value="0.0" step="0.1"></label>
        <label>落下範囲半径 (m): <output type="number" id="result" value="0.0" step="0.1"></label>
    </div>
    <div class="container">
        <canvas id="trajectoryCanvas" width="200" height="500"></canvas>
        <div class="x-axis-label">落下範囲半径 (m)</div>
        <div class="y-axis-label">高度 (m)</div>
    </div>
    <script>
        let canvas = document.getElementById('trajectoryCanvas');
        let ctx = canvas.getContext('2d');

        let input_mass = parseFloat(document.getElementById('input_mass').value);
        let input_ws = parseFloat(document.getElementById('input_ws').value);
        let input_Vx = parseFloat(document.getElementById('input_Vx').value);
        let input_Y = parseFloat(document.getElementById('input_Y').value);
        let input_mu = parseFloat(document.getElementById('input_mu').value);
        let input_rho = parseFloat(document.getElementById('input_rho').value);
        let input_bodylen = parseFloat(document.getElementById('input_bodylen').value);

        let g = 9.81;  // 重力加速度 (m/s^2)

        let dt = 0.001;  // 時間刻み (s)
        let totalTime = 500.0;  // シミュレーション時間 (s)
        let scale = 10; // 1メートルあたりのピクセル数
        //抗力係数の計算
        let calculateCd = (l_rho, l_vel, l_dia, l_mu) => {
            let Re = l_rho*l_vel*l_dia/l_mu;
            let a = 24.0/Re;
            let b = 2.6*(Re/5.0)/(1.0+(Re/5.0)**1.52);
            let c = 0.411*(Re/(2.63*10**5))**(-7.94)/(1.0+(Re/(2.63*10**5))**(-8));
            let d = 0.25*(Re/(10**6))/(1.0+(Re/(10**6)));
            let cd = a + b + c + d;
            return cd;
        }
        // 軌跡の計算
        let calculateTrajectory = (l_mass, l_ws, l_Vx, l_Y, l_mu, l_rho, l_bodylen) => {
            let x = 0.0;
            let y = l_Y;
            let v_x = l_Vx;
            let v_y = 0.0;
            let cd = calculateCd(l_rho, Math.sqrt((v_x - l_ws)**2+v_y**2), l_bodylen, l_mu);
            let k = 0.5 * l_rho * cd * (l_bodylen/2.0)**2 * Math.PI; // 空気抵抗の係数 (kg/m)
            let trajectory = [];
            let y_max = y;
            let t_l = 0;
            let cd_ave = 0;
            let cycle_cnt = 0;


            for (let t = 0; t <= totalTime; t += dt) {

                let v = Math.sqrt((v_x - l_ws) ** 2 + v_y ** 2);
                cd = calculateCd(l_rho, v, l_bodylen, l_mu)
                k = 0.5 * l_rho * cd * (l_bodylen/2.0)**2 * Math.PI;
                let fd = k*v**2
                let fdx = fd * Math.cos(Math.atan2(v_y,v_x - l_ws))
                let fdy = fd * Math.sin(Math.atan2(v_y,v_x - l_ws))
                let dv_xdt = - fdx / l_mass;
                let dv_ydt = -g - fdy / l_mass;

                v_x += dv_xdt * dt;
                v_y += dv_ydt * dt;

                x += v_x * dt;
                y += v_y * dt;

                if(y > y_max){
                    y_max = y;
                }
                trajectory.push({x: x, y: y});
                cycle_cnt = cycle_cnt + 1;
                cd_ave = cd_ave + cd;
                t_l = t;

                if (y < 0) break;  // 地面に到達したらシミュレーション終了
            }

            scale = Math.floor(canvas.height / y_max)
            // scale = 10
            document.getElementById('result').value = x
            document.getElementById('result_cd').value = cd_ave / cycle_cnt
            document.getElementById('result_vel').value = Math.sqrt(v_x**2+v_y**2)
            document.getElementById('result_t').value = t_l
            // alert(Math.sqrt((v_x - l_ws)**2+v_y**2))
            return trajectory;
        }

        // グリッドの描画
        let drawGrid = () => {
            const gridSize = 50; // グリッドの間隔
            ctx.strokeStyle = '#e0e0e0';
            ctx.lineWidth = 1;

            for (let x = 0; x <= canvas.width; x += gridSize) {
                ctx.beginPath();
                ctx.moveTo(x, 0);
                ctx.lineTo(x, canvas.height);
                ctx.stroke();
            }

            for (let y = 0; y <= canvas.height; y += gridSize) {
                ctx.beginPath();
                ctx.moveTo(0, y);
                ctx.lineTo(canvas.width, y);
                ctx.stroke();
            }
        };

        // 軸ラベルの描画
        let drawAxisLabels = () => {
            ctx.font = '12px Arial';
            ctx.fillStyle = 'black';

            // X軸ラベル
            for (let x = 0; x <= canvas.width; x += 50) {
              ctx.fillText((x / scale).toFixed(1), x, canvas.height - 5);

            }

            // Y軸ラベル
            for (let y = 0; y <= canvas.height; y += 50) {
              if(y >= canvas.height){
                ctx.fillText("", 5, y);
              }else{
                ctx.fillText(((canvas.height - y) / scale).toFixed(1), 5, y);
              }
                //ctx.fillText(((canvas.height - y) / scale).toFixed(1), 5, y);
            }
        };

        // 軌跡の描画
        let drawTrajectory = (trajectory) => {

            ctx.clearRect(0, 0, canvas.width, canvas.height);
            drawGrid();
            drawAxisLabels();
            ctx.beginPath();
            ctx.moveTo(0, canvas.height - input_Y * scale);

            trajectory.forEach(point => {
                ctx.lineTo(point.x * scale, canvas.height - point.y * scale); // スケーリング
            });

            ctx.strokeStyle = 'blue';
            ctx.lineWidth = 2;
            ctx.stroke();
            ctx.closePath();
        };

        // フォーム入力を取得して軌跡を更新
        let updateTrajectory = () => {
            input_mass = parseFloat(document.getElementById('input_mass').value);
            input_ws = parseFloat(document.getElementById('input_ws').value);
            input_Vx = parseFloat(document.getElementById('input_Vx').value);
            input_Y = parseFloat(document.getElementById('input_Y').value);
            input_mu = parseFloat(document.getElementById('input_mu').value);
            input_rho = parseFloat(document.getElementById('input_rho').value);
            input_bodylen = parseFloat(document.getElementById('input_bodylen').value);
            // mass = parseFloat(document.getElementById('mass').value);
            // windSpeed = parseFloat(document.getElementById('windSpeed').value);
            // initialVx = parseFloat(document.getElementById('initialVx').value);
            // initialVy = 0.0;
            // initialY = parseFloat(document.getElementById('initialY').value);
            // initialCd = parseFloat(document.getElementById('cd').value);
            // initialrho = parseFloat(document.getElementById('rho').value);
            // initialArea = parseFloat(document.getElementById('area').value);

            trajectory = calculateTrajectory(input_mass, input_ws, input_Vx, input_Y, input_mu, input_rho, input_bodylen);
            drawTrajectory(trajectory);
        }

        // 初期描画
        let initialTrajectory = calculateTrajectory(input_mass, input_ws, input_Vx, input_Y, input_mu, input_rho, input_bodylen);
        drawTrajectory(initialTrajectory);
    </script>
</body>
</html>
