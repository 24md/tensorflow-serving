# Deploy a Machine Learning Model to Production with TensorFlow Serving

The following are the prerequisites and the steps to deploy a Machine Learning Model to production with TensorFlow Serving.

## Prerequisites

* Provision a [Vultr Cloud GPU Server](https://my.vultr.com/deploy/?cloudgpu) using the [NVIDIA NGC](https://www.vultr.com/marketplace/apps/nvidia-ngc/) Marketplace Application

## Set Up the Environment

Create a container using the `tensorflow/tensorflow:latest-gpu-jupyter` image.

```console
# docker run -p 8888:8888 --gpus all -it --rm -v /root/notebooks:/tf/notebooks tensorflow/tensorflow:latest-gpu-jupyter
```

Open the Jupyter Notebook interface on your web browser using the link provided in the output.

Upload the [Fashion MNIST.ipynb](Fashion%20MNIST.ipynb) file to the `notebooks/` folder.

Open the Fashion MNIST notebook and run all the code blocks.

Stop the TensorFlow container using <kbd>Ctrl</kbd> + <kbd>C</kbd>.

You can use the [Fashion MNIST via TF Serving.ipynb](Fashion%20MNIST%20via%20TF%20Serving.ipynb) file to test the TensorFlow Serving functionality by replacing the URL in the `predict()` function on another server or your own machine.

## Serve a Single Model Using TensorFlow Serving

Create a container using the `tensorflow/serving:latest-gpu` image.

```console
# docker run -p 8501:8501 --gpus all -it --rm -v /root/notebooks/saved_models:/models -e MODEL_NAME=fashion_mnist tensorflow/serving:latest-gpu
```

The above command uses the `MODEL_NAME` environment variable to define the name of the model which you want to serve.

Access the server via HTTP using `http://IP:8501/v1/models/MODEL_NAME:FUNCTION`.

## Serve Multiple Models Using TensorFlow Serving

Create a new file named `models.config`.

```console
# nano ~/notebooks/saved_models/models.config
```

Add the following contents to the file and save the file using <kbd>Ctrl</kbd> + <kbd>X</kbd> and <kbd>Enter</kbd>.

```code
model_config_list {
  config {
    name: 'fashion_mnist'
    base_path: '/models/fashion_mnist/'
    model_platform: 'tensorflow'
  }
  config {
    name: 'also_fashion_mnist'
    base_path: '/models/also_fashion_mnist/'
    model_platform: 'tensorflow'
  }
}
```

The above configuration uses each `config{}` block to define a model to serve. You can also use the `model_version_policy` field inside the `config{}` block to serve multiple versions of a model. Refer to [Tensorflow Serving Configuration](https://www.tensorflow.org/tfx/serving/serving_config#serving_a_specific_version_of_a_model) to learn more.

Create a container using the `tensorflow/serving:latest-gpu` image.

```console
# docker run -p 8501:8501 --gpus all -it --rm -v /root/notebooks/saved_models:/models tensorflow/serving:latest-gpu --model_config_file=/models/models.config
```

The above command uses the `--model_config_file` flag to define the path of the `models.config` file inside the container.

Access the server via HTTP using `http://IP:8501/v1/models/MODEL_NAME:FUNCTION`.


## Set Up a Persistent TensorFlow Serving Environment

Create a new directory named `tensorflow-serving`.

```console
# mkdir ~/tensorflow-serving
# cd ~/tensorflow-serving
```

Create a new Docker Compose configuration file.

```console
# nano docker-compose.yaml
```

Add the following contents to the file, replace `YOUR_EMAIL` and `YOUR_DOMAIN` with your actual details and save the file using <kbd>Ctrl</kbd> + <kbd>X</kbd> and <kbd>Enter</kbd>.

```yaml
services:

  serving:
    image: tensorflow/serving:latest-gpu
    restart: unless-stopped
    volumes:
      - "/root/notebooks/saved_models:/models"
    command: --model_config_file=/models/models.config
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: [gpu]

  nginx:
    image: nginx
    restart: unless-stopped
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/dhparam.pem:/etc/ssl/certs/dhparam.pem
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot


  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    command: certonly --webroot -w /var/www/certbot --force-renewal --email YOUR_EMAIL -d YOUR_DOMAIN --agree-tos
```

Ensure that your domain name points to the server using an A record.

Create a new directory for Nginx configuration.

```console
# mkdir nginx
# nano nginx/nginx.conf
```

Add the following contents to the file and save the file using <kbd>Ctrl</kbd> + <kbd>X</kbd> and <kbd>Enter</kbd>.

```nginx
events {}

http {
    server_tokens off;
    charset utf-8;

    server {
        listen 80 default_server;
        server_name _;

        location ~ /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
    }
}
```

Run the `docker-compose` file.

```console
# docker-compose up -d
```

After few minutes, check if the SSL certificate was generated.

```console
# ls certbot/conf/live/YOUR_DOMAIN/
```

Stop the `nginx` container.

```console
# docker-compose stop nginx
```

Swap the `nginx` configuration.

```console
# rm -f nginx/nginx.conf
# nano nginx.conf
```

Add the following contents to the file, replace `YOUR_DOMAIN` with your actual domain and save the file using <kbd>Ctrl</kbd> + <kbd>X</kbd> and <kbd>Enter</kbd>.

```nginx
events {}

http {
    server_tokens off;
    charset utf-8;

    server {
        listen 80 default_server;
        server_name _;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;

        server_name YOUR_DOMAIN;

        ssl_certificate     /etc/letsencrypt/live/YOUR_DOMAIN/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/YOUR_DOMAIN/privkey.pem;

        location / {
            proxy_pass http://serving:8501;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location ~ /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
    }
}
```

Restart the `nginx` container.

```console
# docker-compose restart nginx
```

Access the server via HTTPS using `https://YOUR_DOMAIN/v1/models/MODEL_NAME:FUNCTION`.

Add a new entry in the `crontab`.

```console
# crontab -e
```

Add the following lines to the table.

```cron
0 5 1 */2 *  /usr/local/bin/docker-compose start -f /root/tensorflow-serving/docker-compose.yml certbot
5 5 1 */2 *  /usr/local/bin/docker-compose restart -f /root/tensorflow-serving/docker-compose.yml nginx
```

Exit the `cron` editor using <kbd>Esc</kbd> then <kbd>!wq</kbd> and <kbd>Enter</kbd>.
