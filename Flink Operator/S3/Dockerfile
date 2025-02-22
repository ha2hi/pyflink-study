#
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
# limitations under the License.
################################################################################
# Check https://nightlies.apache.org/flink/flink-docs-master/docs/deployment/resource-providers/standalone/docker/#using-flink-python-on-docker for more details
FROM flink:1.20

# install python3: it has updated Python to 3.9 in Debian 11 and so install Python 3.7 from source, \
# it currently only supports Python 3.6, 3.7 and 3.8 in PyFlink officially.

RUN apt-get update -y && \
apt-get install -y build-essential libssl-dev zlib1g-dev libbz2-dev libffi-dev && \
wget https://www.python.org/ftp/python/3.8.9/Python-3.8.9.tgz && \
tar -xvf Python-3.8.9.tgz && \
cd Python-3.8.9 && \
./configure --without-tests --enable-shared && \
make -j6 && \
make install && \
ldconfig /usr/local/lib && \
cd .. && rm -f Python-3.8.9.tgz && rm -rf Python-3.8.9 && \
ln -s /usr/local/bin/python3 /usr/local/bin/python && \
apt-get clean && \
rm -rf /var/lib/apt/lists/*

# install PyFlink
RUN pip3 install "apache-flink==1.20.0"

# Create S3 Plugin
RUN mkdir /opt/flink/plugins/s3-fs-hadoop && \
    cp /opt/flink/opt/flink-s3-fs-hadoop-1.20.0.jar /opt/flink/plugins/s3-fs-hadoop

USER flink
